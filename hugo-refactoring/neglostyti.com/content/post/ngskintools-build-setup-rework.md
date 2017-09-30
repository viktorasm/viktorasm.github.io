---
title:  "ngSkinTools build setup rework"
date: 2017-09-30
categories: 
    - Maya
    - ngSkinTools
---

For a very long time ngSkinTools was being built with a setup that "just happened". Apache Ant building C++ binaries, python scripts calling python on another `venv`.
 
It's been the part that I've been most afraid to refactor.  


<!--more-->


## What needs to be done.

We want a single button to:
* Build what needs to be built on all operating systems;
* Upon successful build, update sources with new release name: tag Git, update version in changelog, update current version in scripts;
* Create installers;
* Upload binaries online;
* Rebuild and upload our Hugo website with info about new release;

## Review of current setup

My previous build setup was... interesting. I created back in the day when I primarily worked with Java, so I chose the following things:

The main build system was Apache Ant. Build configuration is `ant.xml`. Even today it's a pretty solid build system if you follow it's conventions, but let's agree, you don't want to build multi-platform C++ project with Ant. But I did:

```xml
...
<target name="compile-windows" if="os.windows" depends="vcvars,resolve-windows-parameters">
    <apply executable="${cl.exe}" dir="${build.plugin-outdir}" dest="${build.plugin-outdir}" failonerror="yes" verbose="true" parallel="true">
        <fileset refid="build.sourcefileset" />
        <mapper type="glob" from="*.cpp" to="*.obj" />

        <env key="Path" value="${vcvars.path.flattened}" />
        <env key="include" value="${vcvars.include.flattened}" />

        <arg value="/I" />
        <arg path="${maya.includedir}" />
        <arg value="/I" />
        <arg path="${cpp.lib.boost}" />
        <arg value="/I" />
        <arg path="${build.cpp.include1}" />
        <arg value="/I" />
        <arg path="${build.cpp.include2}" />
        
        <arg line="${compile.platform-args}" />
        <arg value="/DNGSKINTOOLS_PLUGIN_VERSION#\&quot;${project.pluginVersion}\&quot;" />
        <arg value="/DNGSKINTOOLS_WATERMARK#\&quot;${output.buildwatermark}\&quot;" />
        
<!-- ...600lines more stuff like this. -->        
```

Obviously I could not do everything via Ant, so I patched the situation with Python:
```xml
<target name="generate-build-id">
    <exec executable="${python.executable}" outputproperty="output.buildwatermark">
        <arg file="${project.buildscripts}/generate-build-id.py" />
    </exec>
    <echo message="build ID: ${output.buildwatermark}" />
</target>
```

..and Make
```xml
<target name="compile-osx" if="os.osx">
    <exec  executable="make" dir="${project.resources}/make-osx-${mll.maya}" failonerror="true">
        <arg value="-j9"/>
        <env key="MAYA_SDK" value="${maya.devkitlocation}" />
        <env key="MAYA_LOCATION" value="${maya.sdklocation}" />
        <env key="BOOST_SDK" value="${cpp.lib.boost}" />
        <env key="WORKSPACE" value="${build.workspace}/" />
        <env key="BUILD_DIR" value="${build.plugin-outdir}"/>
        <env key="PLUGIN_VERSION" value="${project.pluginVersion}" />
        <env key="PLUGIN_WATERMARK" value="${output.buildwatermark}" />
        <env key="OUTPUT_BINARY" value="${build.pluginfile}"/>
    </exec>
</target>
```


Then everything is glued together via:

```python
def execute(self):
    print "detecting workspace..."
    self.workspace = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
    print "workspace: ", self.workspace

    self.cwd = self.workspace
    
    self.verifyBuildersOnline()
    self.getCurrentVersion()
    self.detectNewVersion()  
    self.verifyNewVersionNewer()
    
    self.updateChangelogHeader()
    self.updateVersionFile()
    self.updatePackageProperties()
    
    self.pushChanges()
    
    if query_yes_no("launch builds?"):
        # runs build for reach OS
        self.launchBuilds()
    if query_yes_no("upload builds?"):
        self.uploadArtifacts()

    # start website server for local edits
    self.runHugoServer()
    self.updateWebsiteMetadata()
    self.generateReleaseEntry()
    self.updateWebsiteChangelog()

    if query_yes_no("publish website?"):
        self.buildAndPublishWebsite()
    
    
    print "\nDone."

```



## Won't a python script work?

Initially I've been tempted to just rewrite the whole thing in plain python to have full control over things. After all, I don't need much, right? Let's summarize.

So, I'd like to:
* Name the build, e.g. 1.7.1;
    * would also be nice to just do a "dry run": build everything but don't publish stuff
* Trigger build C++ binaries:
    * On multiple machines: windows #1 (old build machine that supports maya 2014), windows#2 (new machine that will build most of the new stuff), linux (virtual machine) and osx (my superslow mac mini);
    * Fail the whole build if one mll build fails;
        * E.g. "linux 2014 build fails for your latest code changes you only tested on Windows Maya 2018";
    * Builds are slow so definitely run the in parallel;
        * If things get hairy it would be nice to inspect individual runs in isolation;
    * Maybe run tests for each maya version for each OS;
    * Monitor basic stats (like build times);
    * Collect all binaries into single place for further processing;
* Build python:
    * Run linter;
    * Run various transform scripts;
    * Run integration tests for os/maya version? Only executed manually now on the development machine  
* Update changelog (insert build version and date of the)
* Rebuild documentation
* Build OS specific packages
    * one zip per each environment
    * one OS-specific installer
* Update website with new build info
* Add relevant GIT tags (version)
* Publish builds to AWS:
    * Suspend build, wait for confirmation
* Update website with "new release" page
    * Suspend the build while I add manual content to the release
    * Wait for build resume
* Deploy website
* Stop the build for manual tweaks on the "new build"   
     
     
Also I'd like to:
* Put as much build configuration as possible into code:
    * I'd like to version my build
    * Easier to reproduce build environment if I ever need to rebuild it
* Have some sort of visual feedback; current build of all things takes some good 15 minutes, so in anticipation I can't just have a console of "starting... running... done."
* Treat whole build infrastructure as crash-and-burn:
    * Ability to delete everything, then checkout and rebuild
        * Except for the heavy things like Visual Studio and Maya installations
    * Minimal configuration for the build tool
        * Ideally a single "checkout whole build configuration from git" job
* Maybe build history?... Nice to have.



## Prep work

### CMakeify all binaries
 

### Jenkins slave setup

Configure slaves scripts to run when VMs are executed. 

### Jenkins Pipeline

I chose Jenkins because, as some my colleagues say, "it's affordable". It's not the best CI tool out there, but certainly the most used (basically it's the Javascript of build systems). You get it and it's plugins for free, there is a lot of material online when you need help, etc. The other option I evaluated was GoCD as I've heard a lot of good things about, but haven't worked with before.

So what do we get from Jenkins instead of just running build script on command line?

#### a build UI

You think you don't need one until you want to input a build parameter ("release version") or confirm a non-fatal warning ("distribution v1.2.3 is already uploaded, do you want to override?"). It's just more reassuring to have a tool stare at you.

When you have a more complex build setup which, to your knowledge, will do  all sorts of things, like rewrite files, rewrite tags on git, upload files online, and all kinds of other potentially destructable activities, it's just more comforting to have a better overview of what is going on.

#### Distributed build management

* Build artifacts exchange between different build locations
* Automatic waiting for build nodes to be available; the build will not fail early or in the middle of something if node got disconnected or you just forgot to start a particular VM, you'll just see the "waiting for a node to become online" message in the build log;

#### Overview of the execution
* Breaks down the execution log into individual boxes, so it's convenient to check "what went wrong in the Maya 2014 build on Windows" even though there were a lot of things running in parallel;
* Build statistics. Did some step just got slower? Is something not looking as fast as expected? This could sound like a nice to have (you might say, "build is something I just start and go get some coffee"), but I feel like I have shaved quite some time off the build but having the number stare at me. Like, "prepare virtual environment and run a Python script": 20 seconds. Leave the virtual environment cached between builds: "2 seconds". "Upload files": 2 minutes. Fiddle with parallel uploads: 20seconds. Things like that quickly add up.


## jenkins is "affordable"

Encountered bugs and shortcommings like:
* build silently fails; `archive` task error;
* parallel stages shown as paused: jenkins does not handle them very well; 



## Random tips

* Break down build into individual reusable files:
    * Website redeployment is a part of the build, but I can also update the content and run just the "republish website" script;
    * C++ builds are complicated to set up, therefore I want to run build scripts in isolation to make changes to them;
    * In general, for longer running build you want to be able to launch it's individual parts  in isolation; "only way to test this is running everything" is not a great strategy in this case.


* Make your build scripts very simple
    * Small scope for a task, e.g. "build binary X for 2017 maya on windows"
    * Prefer command line parameters for configuration instead of environment variables or implicit locations on disk, configuration files; 
    * Produce output files to current folder (configure via caller), output messages to stdout/stderr;
    * Integrate via Jenkins
    
You sometimes never know when you have to build more stuff into build. As an example, I realized I need slightly different configuration to build user manual - one for local use, one for online. I implemented that as:
```python
with cwd("userguide"):
    replaceContents('mkdocs.yml','use_directory_urls: true','use_directory_urls: false')
    subprocess.call(['../venv/Scripts/mkdocs.exe', 'build'])
``` 

