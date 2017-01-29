+++
id = "0015"
title = "Gradle : Reduce duplication across projects"
description = "Gradle files contain code, and code generate duplications if not organised and maintaned correctly. Gradle plugins and 'apply from' can help to organise build script code in multiple projects."
date = "2017-01-26T17:40:27+00:00"
author = "luigi"
tags = [ "Gradle", "Android", "Checkstyle" ]
+++

![article-img](/img/blog/0015/scared_tengio_gradle.png)

Holidays are always a good moment to stop and reflect a little bit about the last few projects, and this is even more true for Christmas holidays as November and December are always busy months.

During the last couple of months in Tengio we have created a few open source library and demo apps, and as a reasult we have plenty of duplicated code in build scripts. 

It is time to have a deeper look at the gradle build scripts and plugins that we use and extract the duplication where possible.

I set the following requirements for this task:

* Cost in terms of my time should not be more than 2 days
* The abstraction should not increase the learning curve for other developers in the team
* The changes should affect positively more than 1 project.

## Duplication in gradle scripts

It is quite easy to identify the primary source of duplication in the gradle scripts of our projects. Just by reading through a few project ```build.gradle``` files I have identified the following parts:

* checkstyle
* bintray

## Checkstyle

I hope people reading this blog are familiar with checkstyle. If you are not please google it and try to use it. In the past, it was quite hard and cumbersome to use, but nowdays there are no excuses.

The best way to take advantage of checkstyle is to create a company wide code style rule set that is as close as possible to the current standard style guidelines of you project type. Once the rule set is defined you can enforce the rules in the static code analysis phase of the build scripts and at the same time help developers use the rules with some IDE integration. 

To summarise if you want to use checkstyle, you have to:

* enforce it in your continuous integration
* integrate it in your IDE

Any friction between these two rules and you will have more issues using checkstyle than not using it.

In this case I'm looking at java/android base projects. For this type of projects you can implement checkstyle with the following steps: 

* define the rules of your code style in a file ```checkstyle.xml```.   
* add the checkstyle plugin to your gradle project.
* create a ```codestyle.xml``` file, and import the file in your IDE (at least this should work for android studio and IntelliJ).

## Checkstyle implementation problem

As mentioned before, the codestyle implementation for a project consists of the following files:

* codestyle.xml
* checkstyle.xml
* Plugin repository definition and dependency
* For android project you also need the ridefinition of the checkstyle task so that it can take the proper source code to analyse.

All this is fine for a single project, but as soon as you start to have a few projects you end up with lots of duplications which are expensive to maintain.

## Tengio Checkstyle Plugin

Ok so, we have one problem and we know the requirements for our solution. I think a gradle plugin is the best possible solution in this case. One of the main goals is the ability to share ```checkstyle.xml``` file for all the projects. 

First thing to know: is it actually possible to include and distribute the xml file into the jar and pass it to checkstyle?

A gradle plugin is essentialy a jar. So it is possible to include a file as a resource and read it. More complicated is passing it to checkstyle plugin. With limited time I can't rewrite the checkstyle plugin. I need to have it as a dependency and set the proper configuration to make it work.

Luckily for me the checkstyle plugin has a new ```config``` property that can take the checkstyle Checker configuration as a string.
Finally, with this line of code it is possible to read a file in the resources of the jar as a string  and assign it to the checkstyle plugin:

```
checkstyle.config = project.resources.text.fromString(getClass().getResourceAsStream("checkstyle.xml").text)
```

Ok now I have to learn how to create a gradle plugin, inject a checkstyle plugin and make sure it is configured correctly.

These are the things we need to do:

* 1. create a gradle plugin project
* 2. define a plugin id
* 3. define the plugin class
* 4. Checkstyle plugin dependency
* 5. apply the Checkstyle plugin
* 6. try the plugin 
* 7. add support for android projects
* 8. publish the plugin
* 9. update existing projects to use the new plugin

### 1. Create a gradle plugin project

Note : I will discuss only the necessary parts in the blog, if you want to look at the full code the project is [open source](https://github.com/Tengio/tengio-checkstyle-plugin): 

Simple, just create a build.gradle file with this in your root folder: 

```
apply plugin: 'groovy'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
```

With this in you build.gradle you have a basic groovy gradle plugin (you can also do it in java if you prefer).

### 2. Define a plugin id

Each gradle plugin needs to define a plugin id. The plugin id will also have to associate the id to a Class that implements the Plugin itself.
To do that you have to create a file with name ```[package].[pluginId]```. The file need to be in ```src/main/resources/META-INF``` directory. 
The file needs to contain the class association. Like in this case:

```
implementation-class=com.tengio.gradle.TengioCheckstylePlugin
```

### 3. Define the plugin class

The plugin class is a very simple class in itself that implements Plugin. You can use the apply method to do your own logic.

```
class TengioCheckstylePlugin implements Plugin<Project> {

    void apply(Project project) {
        ....
    }
}
```

### 4. Checkstyle plugin dependency

Dependency declaration with gradle plugins is easy. See here:

```
buildscript {
   ...

   dependencies {
       classpath 'com.puppycrawl.tools:checkstyle:7.3'
   }
}
```

This needs to be placed in the build.gradle of the plugin we are creating.

### 5. Apply the Checkstyle plugin

Going back to the TengioCheckstylePlugin class, to configure and apply the checkstyle plugin to the targeted project you need to use the pluginManager and the apply method as in this script:

```
project.plugins.withType(CheckstylePlugin) {
    project.checkstyle {
        toolVersion = "7.3"
        config = project.resources.text.fromString(getClass().getResourceAsStream("checkstyle.xml").text)
    }
}
project.pluginManager.apply('checkstyle')
```

### 6. Try the plugin 

To test the plugin with a demo project, I have created a simple java project with a couple of classes. There is one tricky part though. I need to publish the plugin somewhere so that my demo app can use it.

Maven to the rescue: 

```
apply plugin: 'maven'

group = 'com.tengio.gradle'
version = '1.0-SNAPSHOT'

uploadArchives {
    repositories {
        mavenDeployer {
            repository(url: mavenLocal().url)
        }
    }
}
```

By adding this to your build.gradle and with the task ```uploadArchives``` you can deploy a version of your plugin locally.
In the demo project you can easily pick up the plugin by adding usual plugin declaration:

```
buildscript {
    repositories {
        ...
        mavenLocal()
    }
    dependencies {
        classpath 'com.tengio.gradle:tengio-checkstyle-plugin:1.0-SNAPSHOT'
    }
}

apply plugin: 'com.tengio.gradle.tengio-checkstyle-plugin'
```

Ok now everything works on a normal java project. 

### 7. Add support for android projects

When you apply the checkstyle plugin to a java project, gradle will automatically attach two tasks *checkstyleMain* and *checkstyleTest* as dependencies of the check task.

With android though this doesn't happen. This is probably related to the different way android plugin manager defines the sources paths of the project. If you have used checkstyle on android project you already know this. Anyway we basically have to do that manually:

```
if(!hasTask(project, 'checkstyleMain')) {
    Checkstyle c = project.tasks.create('checkstyleMain', Checkstyle)
    c.source = 'src'
    c.include '**/*.java'
    c.exclude '**/gen/**'
    c.classpath = project.files()
    c.config = project.resources.text.fromString(getClass().getResourceAsStream("checkstyle.xml").text)
    project.tasks.getByName('check').dependsOn 'checkstyleMain'
}
```

The interesting bit is the last line:
```
project.tasks.getByName('check').dependsOn 'checkstyleMain'
```
It is adding the newly defined task as a Dependency of 'check'. Check is one of the goals executed during an android gradle build.

### 8. Publish the plugin

I'm used to publish Tengio's open source project to bintray, and it is possible to publish a gradle plugin to bintray. However, I noticed that there is a specific gradle repository for plugins. So I gave it a go.

Go to this link and follow the steps, this pubblication process is actually very simple (one of the best I have used so far).

Once it is published you can use it like any other public plugin:

```
buildscript {
    repositories {
        ...
        maven {
            url "https://plugins.gradle.org/m2/"
        }
    }
    dependencies {
        ...
        classpath "com.tengio.gradle:tengio-checkstyle-plugin:1.0"
    }
}

apply plugin: 'com.tengio.gradle.tengio-checkstyle-plugin'
```

### 9. Update existing projects to use the new plugin

Finally, the best part and is all about deleting stuff.


## Bintray and maven

At Tengio we have a few open source projects. All open source projects should be as easy as possible to use. For this reason all our projects are published on Bintray. 

Bintray requires the maven plugin to prepare some meta-information of the artifact. Also for Android projects, bintray needs some custom configuration. See this example: 

```
apply plugin: 'com.jfrog.bintray'
apply plugin: 'com.github.dcendents.android-maven'

project.group = "com.tengio.android"
project.afterEvaluate {
    project.version = android.defaultConfig.versionName
}

bintray {
    user = ''
    key = ''
    if (project.hasProperty('BINTRAY_USER') && project.property('BINTRAY_KEY')) {
        user = project.property('BINTRAY_USER')
        key = project.property('BINTRAY_KEY')
    }
    configurations = ['archives']
    pkg {
        repo = 'maven'
        userOrg = 'tengioltd'
        licenses = ['Apache-2.0']
    }
}

android.libraryVariants.all { variant ->
    def name = variant.buildType.name
    def task = project.tasks.create "jar${name.capitalize()}", Jar
    task.dependsOn variant.javaCompile
    task.from variant.javaCompile.destinationDir
    artifacts.add('archives', task);
}

task sourcesJar(type: Jar) {
    from android.sourceSets.main.java.srcDirs
    classifier = 'sources'
}

task javadoc(type: Javadoc) {
    source = android.sourceSets.main.java.srcDirs
    failOnError false
}

task javadocJar(type: Jar, dependsOn: javadoc) {
    classifier = 'javadoc'
    from javadoc.destinationDir
}

artifacts {
    archives javadocJar
    archives sourcesJar
}
```

**NOTE:** Note the trick that we use to avoid the duplication of the versionName:
```
project.afterEvaluate {
    project.version = android.defaultConfig.versionName
}
```

## Bintray and maven implementation problem 

You can see the problem already. For all the Android projects we have to write the previous snippet of code. 


## Tengio Bintray Script

Digging stuff on how to use checkstyle took me quite a bit and having seen the difficulties, I'm afraid I can't follow the same plugin solution I used for checkstyle.

Luckily for me there is another clever solution for gradle to reuse code: ```apply from: url```. 

With this you can apply a script to an existing build. It is quite awesome. There is only one drawback though, the ```apply from:``` is executed after the evaluation phase. This means that it is not possible to add dependencies from within the applyed script.

Still a pretty good result. We remove all the code making sure to leave the plugin dependencies:

```
buildscript {
  repositories {
    ...
    dependencies {
        ...
        classpath 'com.jfrog.bintray.gradle:gradle-bintray-plugin:1.7.3'
        classpath 'com.github.dcendents:android-maven-gradle-plugin:1.5'
    }
}
```
And we apply the script directly from the github repository:
```
apply from: 'https://raw.githubusercontent.com/Tengio/tengio-bintray-script/master/bintray.gradle'
```

## Conclusions

Overall I'm happy with checkstyle plugin. Also the bintray imlementation is much better than before. I know the ideal solution would be a plugin for bintray too, but I leave it for another holiday.

To be honest all this was also an elaborate excuse to get to build some plugin with gradle. I always wanted to try it.

![article-img-centered](/img/blog/0015/gradle_tengio_friends.png)