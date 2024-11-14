+++
author = "hannibal"
categories = ["devops", "problem solving"]
date = "2015-10-02"
tags = ["jenkins"]
title = "How to Aggregate Tests with Jenkins with Aggregate Plugin on non-relating jobs"
url = "/2015/10/02/how-to-aggregate-tests-with-jenkins-with-aggregate-plugin-on-non-relating-jobs/"

+++

Hello folks.

Today, I would like to talk about something I came in contact with, and was hard to find a proper answer / solution for it.

So I'm writing this down to document my findings. Like the title says, this is about aggregating test result with Jenkins, using the plug-in provided. If you, like me, have a pipeline structure which do not work on the same artifact, but do have a upstream-downstream relationship, you will have a hard time configuring and making Aggregation work. So here is how, I fixed the issue.

# Connection

In order for the aggregation to work, there needs to be an **artifact connection** between the upstream and downstream projects. And that is the key. But if you don't have that, well, let's create one. I have a parent job configured like this one. =>

~~~xml

<?xml version='1.0' encoding='UTF-8'?>
<project>
  <actions/>
  <description></description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.scm.NullSCM"/>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders/>
  <publishers>
    <hudson.tasks.test.AggregatedTestResultPublisher plugin="junit@1.9">
      <includeFailedBuilds>false</includeFailedBuilds>
    </hudson.tasks.test.AggregatedTestResultPublisher>
    <hudson.tasks.BuildTrigger>
      <childProjects>ChildJob</childProjects>
      <threshold>
        <name>SUCCESS</name>
        <ordinal></ordinal>
        <color>BLUE</color>
        <completeBuild>true</completeBuild>
      </threshold>
    </hudson.tasks.BuildTrigger>
  </publishers>
  <buildWrappers/>
</project>
~~~

As you can see, it's pretty basic. It isn't much. It's supposed to be a trigger job for downstream projects. You could have this one at anything. Maybe scheduled, or have some kind of gathering here of some results, and so on and so forth. The end part of the configuration is the interesting bit.

Aggregation is setup, but it won't work, because despite there being an upstream/downstream relationship, there also needs to be an artifact connection which uses **fingerprinting**. Fingerprinting for Jenkins is needed in oder to make the physical connection between the jobs via hashes. This is what you will get if that is not setup:

But if there is no artifact between them, what do you do? You create one.

# The Artifact which Binds Us

Adding a simple **timestamp file** is enough to make a connection. So let's do that. This is how it will look like =>

The important bits about this picture are the small echo which simply creates a file which will contain some time stamp data, and after that the archive artifact, which also fingerprints that file, marking it with a hash which identifies this job as using that particular artifact.

Now, the next step is to create the connection. For that, you need the artifact copy plugin => <a href="https://wiki.jenkins-ci.org/display/JENKINS/Copy+Artifact+Plugin" target="_blank">Copy Artifact Plugin</a>.

With this, we create the childs configuration like this:

~~~xml

<?xml version='1.0' encoding='UTF-8'?>
<project>
  <actions/>
  <description></description>
  <keepDependencies>false</keepDependencies>
  <properties/>
  <scm class="hudson.plugins.git.GitSCM" plugin="git@2.4.0">
    <configVersion>2</configVersion>
    <userRemoteConfigs>
      <hudson.plugins.git.UserRemoteConfig>
        <url>https://github.com/Skarlso/DataMung.git</url>
      </hudson.plugins.git.UserRemoteConfig>
    </userRemoteConfigs>
    <branches>
      <hudson.plugins.git.BranchSpec>
        <name>*/master</name>
      </hudson.plugins.git.BranchSpec>
    </branches>
    <doGenerateSubmoduleConfigurations>false</doGenerateSubmoduleConfigurations>
    <submoduleCfg class="list"/>
    <extensions/>
  </scm>
  <canRoam>true</canRoam>
  <disabled>false</disabled>
  <blockBuildWhenDownstreamBuilding>false</blockBuildWhenDownstreamBuilding>
  <blockBuildWhenUpstreamBuilding>false</blockBuildWhenUpstreamBuilding>
  <triggers/>
  <concurrentBuild>false</concurrentBuild>
  <builders>
    <hudson.plugins.gradle.Gradle plugin="gradle@1.24">
      <description></description>
      <switches></switches>
      <tasks>assemble check</tasks>
      <rootBuildScriptDir></rootBuildScriptDir>
      <buildFile>build.gradle</buildFile>
      <gradleName>(Default)</gradleName>
      <useWrapper>true</useWrapper>
      <makeExecutable>false</makeExecutable>
      <fromRootBuildScriptDir>true</fromRootBuildScriptDir>
      <useWorkspaceAsHome>false</useWorkspaceAsHome>
    </hudson.plugins.gradle.Gradle>
    <hudson.plugins.copyartifact.CopyArtifact plugin="copyartifact@1.36">
      <project>ParentJob</project>
      <filter>timestamp.data</filter>
      <target></target>
      <excludes></excludes>
      <selector class="hudson.plugins.copyartifact.TriggeredBuildSelector">
        <upstreamFilterStrategy>UseGlobalSetting</upstreamFilterStrategy>
      </selector>
      <doNotFingerprintArtifacts>false</doNotFingerprintArtifacts>
    </hudson.plugins.copyartifact.CopyArtifact>
  </builders>
  <publishers>
    <hudson.tasks.junit.JUnitResultArchiver plugin="junit@1.9">
      <testResults>build/test-results/*.xml</testResults>
      <keepLongStdio>false</keepLongStdio>
      <healthScaleFactor>1.0</healthScaleFactor>
    </hudson.tasks.junit.JUnitResultArchiver>
  </publishers>
  <buildWrappers>
    <hudson.plugins.ws__cleanup.PreBuildCleanup plugin="ws-cleanup@0.28">
      <deleteDirs>false</deleteDirs>
      <cleanupParameter></cleanupParameter>
      <externalDelete></externalDelete>
    </hudson.plugins.ws__cleanup.PreBuildCleanup>
  </buildWrappers>
</project>
~~~

Again, the improtant bit is this:

After the copy is setup, we launch our parent job and if everything is correct, you should see something like this:

# Wrapping it Up

For final words, important bit to take away from this is that you need an **artifact connection between the jobs** to make this work. Whatever your downstream / upstream connection is, it doesn't matter. Also, there can be a problem that you have everything set up, and there are artifacts which bind the jobs together but you still can't see the results, then your best option is to specify the jobs BY NAME in the aggregate test plug-in like this:

I know this is a pain if there are multiple jobs, but at least, jenkins is providing you with Autoexpande once you start typing.

Of course this also works with multiple downstream jobs if they copy the artifact to themselves.

Any questions, please feel free to comment and I will answer to the best of my knowledge.

Cheers,
Gergely.
