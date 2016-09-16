# Continuous Integration

This page describes how build Spoofax langauges on a Jenkins buildfarm.

Full continous integration includes:

* Building the language on the buildfarm
* Running SPT tests as part of the build
* Publishing an eclipse updatesite for the language
* Doing the above on every commit, and on every spoofax-master update

Setting up continues integration is a two step process.
The first step is to setup a local maven build for building the language, running the tests and creating an update site.
The second step is configuring Jenkins to perform these maven builds and publish the artifacts.

## Local Maven Build

The local Maven build starts from the generate new project wizard (you need the generated files).
`New Project` > `New Spoofax language project` > select all generation options. This generates 6 projects in total.

(Move the six projects to a new folder.)
Create a parent pom.xml in this folder:

```
<?xml version="1.0" encoding="UTF-8"?>
<project
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd"
  xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <modelVersion>4.0.0</modelVersion>
  <artifactId>Entity.build</artifactId>
  <version>0.1.0-SNAPSHOT</version>
  <packaging>pom</packaging>

  <parent>
    <groupId>org.metaborg</groupId>
    <artifactId>parent</artifactId>
    <version>2.1.0-SNAPSHOT</version>
    <relativePath />
  </parent>

  <modules>
    <module>Entity</module>
    <module>Entity.eclipse</module>
    <module>Entity.eclipse.feature</module>
    <module>Entity.eclipse.site</module>
    <module>Entity.test</module>
  </modules>

  <repositories>
    <repository>
      <id>metaborg-release-repo</id>
      <url>http://artifacts.metaborg.org/content/repositories/releases/</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
    <repository>
      <id>metaborg-snapshot-repo</id>
      <url>http://artifacts.metaborg.org/content/repositories/snapshots/</url>
      <releases>
        <enabled>false</enabled>
      </releases>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </repository>
    <repository>
      <id>spoofax-eclipse-repo</id>
      <url>http://download.spoofax.org/update/nightly/</url>
      <layout>p2</layout>
      <releases>
        <enabled>false</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>

  <pluginRepositories>
    <pluginRepository>
      <id>metaborg-release-repo</id>
      <url>http://artifacts.metaborg.org/content/repositories/releases/</url>
      <releases>
        <enabled>true</enabled>
      </releases>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </pluginRepository>
    <pluginRepository>
      <id>metaborg-snapshot-repo</id>
      <url>http://artifacts.metaborg.org/content/repositories/snapshots/</url>
      <releases>
        <enabled>false</enabled>
      </releases>
      <snapshots>
        <enabled>true</enabled>
      </snapshots>
    </pluginRepository>
  </pluginRepositories>

</project>
```

Copy the `.mvn` folder from your language folder to the parent folder. e.g. `Entity/Entity/.mvn -> Entity/.mvn`.
(Error message otherwise: `[ERROR] Failed to execute goal org.metaborg:spoofax-maven-plugin:2.1.0-SNAPSHOT:clean (default-clean) on project Entity: Building clean input failed unexpectedly: Language for dependency org.metaborg:org.metaborg.meta.lang.esv:2.1.0-SNAPSHOT does not exist -> [Help 1]`)

Fix the generated test yaml file (known issue) e.g. `Entity/Entity.test/metaborg.yaml`.
(Error message otherwise: `[ERROR] Field 'id' must be set`)

```
---
dependencies:
  compile:
  - org.example:Entity:0.1.0-SNAPSHOT
  - org.metaborg:org.metaborg.meta.lang.spt:${metaborgVersion}
```
to
```
---
id: org.example:Entity.test:0.1.0-SNAPSHOT
name: Entity
metaborgVersion: 2.1.0-SNAPSHOT
dependencies:
  compile:
  - org.example:Entity:0.1.0-SNAPSHOT
  - org.metaborg:org.metaborg.meta.lang.spt:${metaborgVersion}
build:
  useBuildSystemSpec: true
```

The maven build should now succeed:

```
[INFO] ------------------------------------------------------------------------
[INFO] Building Entity.build 0.1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:3.0.0:clean (default-clean) @ Entity.build ---
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary:
[INFO] 
[INFO] Entity ............................................. SUCCESS [ 31.033 s]
[INFO] Entity.eclipse ..................................... SUCCESS [  1.252 s]
[INFO] Entity.eclipse.feature ............................. SUCCESS [  0.469 s]
[INFO] Entity.eclipse.site ................................ SUCCESS [  3.776 s]
[INFO] Entity.test ........................................ SUCCESS [  0.140 s]
[INFO] Entity.build ....................................... SUCCESS [  0.013 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 51.354 s
[INFO] Finished at: 2016-09-16T16:58:25+02:00
[INFO] Final Memory: 280M/963M
[INFO] ------------------------------------------------------------------------
```

## Build on Jenkins

`New Item` > enter a name and choose `Multibranch Pipeline`

Add the git repo `Branch Sources` > `Add source` > `Git`. Fill in project repository such as `https://github.com/MetaBorgCube/metaborg-entity.git`, select credentials, and save.

You should now get a message saying that the repository has branch but does not meet the criteria, as the `Jenkinsfile` is not setup yet.

Create the a file `Jenkinsfile` in the root of the repository containing (be sure to update the update site path):

```
// workaround for branch-names which contain a slash
def getWorkspace() {
    pwd().replace("%2F", "_")
}

node{
  ws(getWorkspace()) {
    stage 'Build and Test'
    checkout scm
    sh "git clean -fXd" // make sure generated files are removed (git-ignored files). Use "-fxd" to also remove untracked files, but note that this will also remove .repository forcing mvn to download all artifacts each build
    withMaven(
      mavenLocalRepo: '.repository',
      mavenOpts: '-Xmx1024m -Xss16m'
    ){
      // Run the maven build
      sh "mvn -B -U clean verify -DforceContextQualifier=\$(date +%Y%m%d%H%M) "
    }
    archiveArtifacts artifacts: 'icedust.eclipse.updatesite/target/site/', excludes: null, onlyIfSuccessful: true
  }
}
```

Go to the Jenkins project > `Branch Indexing` > `Run now`. This should trigger the build of the master branch.

In order to trigger Jenkins to build on every commit we need to install a GitHub service.
In the GitHub repository go to `Settings` > `Integrations & services` > `Add service` > `Jenkins (Git plugin)` (not GitHub plugin) and provide the jenkins url (for example http://buildfarm.metaborg.org/ )

For a GitHub build-badge add the following the the readme file:
```
[![Build status](http://buildfarm.metaborg.org/job/Entity/job/master/badge/icon)](http://buildfarm.metaborg.org/job/Entity/job/master/)
```

To trigger the build of specific branch as regression test after spoofax-master is built we need to add the trigger to the spoofax-master job (use with caution!).
Go to the bottom `Build` > `Add build step` > `Trigger/call builds on other projects` and specify the project to build e.g. `Entity/master`.
(Currently there is no way for a multibranch pipeline project to specify a build trigger at the project itself rather than at the spoofax-master project.)

TODO: figure out how to use `Promoted Builds` to promote spoofax-master only if language build succeeds.