---
layout: post
title:  "Continuous Release With Maven"
date:   2015-06-28 10:18:46
categories: CR
comments: true
---


<img style="float: left;margin-right:20px;" src="/assets/2015-06-28-continuous-release-with-maven/maven_logo.png">


Continuous Release process allows an artifact to be continuously **releasable** and promotable to the upper environment. 
  
The topic has been covered by a large amount of articles, so why this new one? Continuous release of an Application (eg. executable jar, war)
   is indeed well covered but not the continuous release of the dependencies (eg. jar library) of an Application.

In this article we will cover both topics and we will get ride of the release pipeline I describe in my previous article `Jenkins Workflow - Pipeline de release` to 
rely solely on a slightly modified continuous integration pipeline. I will end the article with a description of the Continuous Integration pressure paradigm.

   
<!--more-->


# ToC
* TOC
{:toc}


# Before few rules must be followed

Because a non snapshot build must be reproducible no modification are allowed once a project is released. This include no modification of the project and no modification
  of the dependencies. Thus project version must be fixed and version dependencies too


# Scenario

We have two projects: **Application** and **Dependency**. Developments of **Application** and **Dependency** are coupled for an iteration 
because **Application** needs a feature of **Dependency**.

* **Application** 1.0.0-SNAPSHOT

{% highlight xml linenos %}
<project>
    <groupId>org.nlab.article.release</groupId>
    <artifactId>application</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</project>
{% endhighlight %}

* **Dependency** 1.2.0-SNAPSHOT

{% highlight xml linenos %}
<project>
    <groupId>org.nlab.article.release</groupId>
    <artifactId>dependency</artifactId>
    <version>1.2.0-SNAPSHOT</version>
</project>
{% endhighlight %}

**Application** depends on **Dependency**:

{% highlight xml linenos %}
<dependency>
    <groupId>org.nlab.article.release</groupId>
    <artifactId>dependency</artifactId>
    <version>1.2.0-SNAPSHOT</version>
</dependency>
{% endhighlight %}


# The process

The target process is a slightly modified continuous integration pipeline:

* Fix versions:
  * Project version
  * Dependency versions
* Compile, Test, Package, Deploy
* If the project passes all the tests:
  * Promote the project artifact (ie. deploy to the upper environment)
  * Increase the project version


This process does not involve the common release steps: no need to remove snapshot qualifier, commit and wait
for the jenkins pipeline to finish. The usual continuous integration pipeline is self sufficient and generate a releasable
 artifact.
  


# Fix **Application** and **Dependency** versions

## Using a Build Number

To fix the version I will use the build number for three reasons: 

* It is a best practice to deploy only once a release version.
* A build number relates to a meaningful information (jenkins build number, SCM revision...).    
* It is not always possible to use the incremental or minor part of the version to do continuous release.

## And the plugin versions

Before compilation we fix the version using the `versions:set` goal:

{% highlight bash linenos %}
mvn build-helper:parse-version versions:set -DnewVersion=${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.incrementalVersion}-${BUILD_NUMBER}
{% endhighlight %}

Where [BUILD_NUMBER](http://www.mojohaus.org/versions-maven-plugin/version-rules.html) can be a number coming from the SCM or the Jenkins Job build number. 
A build must have a BUILD_NUMBER greater than the previous builds.

`build-helper:parse-version` is a convenient method to extract the version component.

**Application and Dependency versions are now fixed.**
 
 
# Fix **Application** dependency version
 
## The available options
 
* [`versions:use-releases`](http://www.mojohaus.org/versions-maven-plugin/use-releases-mojo.html): *searches the pom for all -SNAPSHOT versions which have been released and replaces them with the corresponding release version.* 

    Only update version using version without qualifier/build number. 
    Not suitable. 

* [`versions:use-latest-releases`](http://www.mojohaus.org/versions-maven-plugin/use-latest-releases-mojo.html): *searches the pom for all versions which have been a newer version and replaces them with the latest version.* 

    Update version using latest release. Update scope is configurable (eg. [`allowIncrementalUpdates`](http://www.mojohaus.org/versions-maven-plugin/use-latest-releases-mojo.html#allowIncrementalUpdates).
    May be useful if continuous release is done using the incremental version.
 
* [`versions:resolve-ranges`](http://www.mojohaus.org/versions-maven-plugin/resolve-ranges-mojo.html): *finds dependencies using version ranges and resolves the range to the specific version being used.*

    Suitable if **Dependency** version is defined as a range.

* [`versions:update-properties`](http://www.mojohaus.org/versions-maven-plugin/update-properties-mojo.html): *updates properties defined in a project so that they correspond to the latest available version of specific dependencies. This can be useful if a suite of dependencies must all be locked to one version.*

    Suitable if **Dependency** version property is defined as a range. As we see, this goal add another level of flexibility.

Note: All goals does not supports Maven properties defined in the root pom and used in a module. For this case `dependencyManagement` may be used in the root pom.


## Range to the rescue

The only suitable option is the `versions:resolve-ranges` goal. **Application** must depend on **Dependency** using a 
[version range](https://maven.apache.org/enforcer/enforcer-rules/versionRanges.html) instead of a SNAPSHOT.
In our case the range is `[1.2.0,1.2.0-99999]`:

{% highlight xml linenos %}
<dependency>
    <groupId>org.nlab.article.release</groupId>
    <artifactId>dependency</artifactId>
    <version>[1.2.0,1.2.0-99999]</version>
</dependency>
{% endhighlight %}


The **Application** now depends only on deployed version of **Dependency** and no longer depends on a SNAPSHOT dependency.

To fix the range we call the `versions:resolve-ranges` goal:

{% highlight bash linenos %}
#> mvn versions:resolve-ranges
{% endhighlight %}

The dependency is now:

{% highlight xml linenos %}
<dependency>
    <groupId>org.nlab.article.release</groupId>
    <artifactId>dependency</artifactId>
    <version>1.2.0-123</version>
</dependency>
{% endhighlight %}


**The dependency version is now fixed.**


# Continuous Integration Pressure

The Continuous Integration Pressure concept is simple: When a new version of **Dependency** is deployed it must trigger the pipeline of **Application** in order to
test the integration of **Dependency** into **Application**

The concept could go further, **Application** must test the integration prior to integrating the new version of **Dependency**. If the test failed, the integration
should be reviewed but must not block the developers or the pipeline.


On the CI side, a new version of **Dependency** will automatically trigger the integration test of all **Application** that depend on it.

There are different levels:

* **Application** depends on **Dependency** using a version range
* **Application** depends on **Dependency** using a fixed version

## Using a version range

On the CI side, a new version of **Dependency** will automatically trigger the **Application** pipeline. As **Application** is using a range, the build will
use the latest build number. No modification of the above process is needed.

## Using a fixed version

The drawback of SNAPSHOT and version range is their volatile nature. A build or the tests of **Application** may failed because of
 a new version of **Dependency**. A fixed version resolves this issue.
 
On the CI side, a new version of **Dependency** will automatically trigger the **Application** integration pipelines. 
This integration pipeline :

* Updates the fixed version of **Dependency** 
* Launches the test
* If tests are ok, the version change is committed 

It involves some modification of the process.

### Continuous release without build number

if the incremental or minor part of the version is used to do continuous release, the **Dependency** version could be updated using `versions:use-latest-releases`
and its properties (eg. `allowIncrementalUpdates`). That's it the version is updated.


### Continuous release with build number

Vincent Latombe [on the CastCodeur mailing list](https://groups.google.com/d/msg/lescastcodeurs/yig2NTbr6vo/l1jxrLV5wHsJ) suggests me this process (in french). 

If a range is used it involves more configuration. We should retain the range somewhere in the POM to be able to fix the **Dependency** version.
  
For this purpose we use the `versions:update-properties` goal using the `properties` property which allows to
add restrictions that apply to specific properties. 
We create a `dependency.version` maven property which holds the current version of **Dependency**.  
**Dependency** version is set to this property.

{% highlight xml linenos %}
    <properties>
        <dependency.version>1.2.0-123</dependency.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.nlab.article.release</groupId>
            <artifactId>dependency</artifactId>
            <version>${dependency.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>versions-maven-plugin</artifactId>
                <version>2.2</version>
                <configuration>
                    <properties>
                        <property>
                            <name>dependency.version</name>
                            <version>[1.2.0,1.2.0-9999]</version>
                        </property>
                    </properties>
                </configuration>
            </plugin>
        </plugins>
    </build>
{% endhighlight %}

When we execute the goal:
{% highlight bash linenos %}
mvn versions:update-properties
{% endhighlight %}

The versions plugin updates `dependency.version` using the provided restrictions range.



# Conclusion

We get continuous release for application and dependencies with continuous integration pressure. 
Every artifact is releasable without the burden of a release process. 
The same pipeline can be used to do CI and to generate a promotable artifact.  
  




