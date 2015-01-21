---
layout: post
title:  "Automated testing and Maya plugin development"
date:   2015-01-01 20:00:00
categories: TDD Maya
---

Writing another article on automated testing might seem silly at first. TDD is exhaustively discussed already, and there's plenty of good material about it already, why another?

I thought that instead of writing a tutorial, or a guide of some sort, I just want to share my story, and thought process of transitioning to TDD setup in [ngskintools.com](http://www.ngskintools.com) project.

I'll probably be skipping quite a few technical details on how this or that is implemented. I will assume that the reader who's interested in such a topic would have some necessary expertise to fill in the gaps.

## The problem

During the first couple of years or so, ngSkinTools development workflow was nothing short of a mess, and could be summarized as:
 
<p class="graf--pullquote">I have some tests, but some things are easier to test by hand. I’m kind-of managing it this way.</p>

There would be ad-hock shell buttons in Maya to launch tests, reload plugin binary or python code, and various other utilities. Various "useful" code snippets would be kept in script editor to run inside Maya, for various things I would be working on at the moment, from adjusting python path, to setting up a test scene or launching a particular sequence of commands while I'm debugging something.

Not a lot would be automated, so coding/testing workflow would generally mean: code, build, alt-tab (or worse, relaunch maya if it crashed from previous run), load test scene, perform some actions in UI, observe result, repeat.

C++ plugin output went to Maya standard output, Python output went to Script Editor history; tracking "what's going on" during execution meant a lot of hit and miss.

Putting project aside and getting back to it after a couple of months would require quite a lot of fiddling around to get things to build again, run stuff in debug mode, figure out how to test this or that. In the even of having to rebuild my development environment, preparing Maya for such workflow would require quite some manual fiddling around, getting all required paths and buttons setup again.  

This does sound really amature, I know. I guess the reason I ended up in this kind-of state was mainly discouraging thoughts like:

*"It's not really clear how you can run tests in Maya"*

*"I don't quite know how I will automate UI tests"*

*"Poor IDE support for such type of testing"*

*"I already write tests though... just some things are easier to test by hand"*

*"I'm kind-of managing it this way"*

## The goal
The whole thing needed ... structure. A few things are obvious:

* Setup the project to enable *comfortable* test-driven development: write tests for features under development, reproduce defects as tests, etc.; adding, writting and executing tests should be *simple* enough not to be discouraging; 
* Eliminate the need of "usefull snippets" and disposable code: wrap utility code into tests, automate or rethink the use case in general;
* Reduce the number of actions needed to rerun tests to a minimum. No manual managing of reloading plugin binaries or python code; all prerequisites should be handled by test runner automatically. 
* Remove the need to switch to Maya for test execution: code, run tests and inspect results within IDE, just like for other type of development projects;
* Unify debug outputs: ideally, single log for C++, Python, test outputs, so that the timeline of events is clearer; 


But most importantly of all:

<p class="graf--pullquote">adding, writting and executing tests should be simple enough not to be discouraging.</p>

In other types of projects I'm working on (mostly web-related Java, Python, Node.js, etc), there's a strong support for this. You write a test and it's discovered and added to the test suite automatically. Your tests execute as part of a build.  Major frameworks make it easy to test code written against it. IDE's let you run the whole suite, or just the test you're currently working on at the moment. Supporting comfortable TDD is a focus of any decent platform out there.

Can't say that about Maya. 

From C++ API side, you can't mock much; the way you have to use API binds you to it. A good example would be `MPxCommand.doIt(const MArgList& args)`; you can't test the method because there's no way to instance `MArgList`. Except for cases where you can completely separate algorithm from Maya API, you'll need to have Maya running, either in it's normal form or as standalone library. Even for something as simple as using a data structure with a MString. 

For Python side, you'd really wish there was a mechanism to trigger UI events, force processing of script jobs in the middle of your tests, etc. In other words, except for built-in Python tools, you're on your own.

Not saying that the problems cannot be adressed; it's just that it's not trivial to overcome, and might be the reason why test-driven development is rarely ever discussed in Maya context.

## The result

Let's overview where I am now. It's not really a one-click solution yet, but comfortable enough for me not to worry at this point.

Primary platform for development became Linux. It's been a few years I'm not using Windows for work, and really enjoying the benefits of a developer-friendly OS. Switching ngSkinTools dev setup over to Linux was always on my wishlist and I'm glad I finally got it out of the way. 

I've migrated coding activities to single IDE, which is Eclipse. Both C++ and Python side of plugin is written there, and also all other types of things, like ngSkinTools update check server, or this very blog.

Maya is setup to launch from within IDE. It's not just launching - the utility fully configures Maya to be ready for test execution. A special workspace where Maya is looking for shelves, scripts, startup options is created from scratch, so that the application configuration is *identical* each time the tests are executed. Console output is redirected to Eclipse.

There are two major test suites to execute, both having a button to launch from IDE. 

C++ test suite, based on [googletest](https://code.google.com/p/googletest/), is just a simple executable created as a part of the debug build. Internally it loads up Maya standalone, and executes tests against that container, so no "real" Maya involvement here. 

Python test suite is actually executed within Maya. When you click a button in IDE, a command to run the whole test suite is sent to Maya, and when it completes, test output is shown back in IDE. 

This all combines into a nice, handle-everything-in-one-tool setup:

<div style="margin: 0 -100px; text-align: center;">
<img src="/assets/maya-test/ide-overview.png" />
</div>



----
----
----

## Pick up the usefull pieces from below sections and discard the rest

So this is where I am now. It's not really a one-click solution yet, but comfortable enough for me not to worry at this point.

### Running tests

The typical development session would start with running the whole set of tests. First, let's fire up Maya, there's a shortcut in IDE prepared for that; Maya quietly starts in the background, gets configured with all sorts of specific options just for this project, and starts listening for test launches.

Then, I execute the whole test suite. After it completes, I get this:
![Test result](/assets/maya-test/test-result.png)

The console on the left is test execution script, which is mostly just shows test summary. The console on the right is Maya: it shows both C++ `cout`s and Python `print`s, and for each test being performed, the separator is put so I can see where one test ends and another begins, if any troubleshooting is needed.

If any test fails, the output would similar to this, with a convenient hyperlink to exact place in the code, being another benefit of doing everything in one tool:
![Test result](/assets/maya-test/test-result-fail.png)

### Getting all tests together

There's no obvious way of "run all tests that you find" from inside Maya, things like [Nose](http://pythontesting.net/framework/nose/nose-introduction/) don't quite work; my solution is to use a highly simplified version of Nose approach, by going through all of modules and finding any test class. I do it with a `testsAll.py` in root test package with this:

{% highlight python %}
for root, dirds, files in os.walk(os.path.dirname(__file__)):
    root = root[len(packageRoot)+1:].replace(os.sep,".")
    
    for f in files:
        if f in ["__init__.py","testsAll.py","testUtils.py","reloadPlugin.py"]:
            continue
        if f.endswith(".py"):
            module = __import__(root+"."+f[:-3],fromlist=[root])
            for _,c in inspect.getmembers(module, inspect.isclass):
                if isTestsClass(c):
                    decorateTestMethods(c)
                    globals()[c.__name__.replace(".","_")] = c
{% endhighlight %}  

Basically, after imported, `testsAll` module will have imported all tests classes that it finds in subpackages. `decorateTestMethods` adds some additional behaviour like printing test name in the log before each test.

Having this `testsAll` module means that I can use `unittest.TestLoader().loadTestsFromModule` to programmatically load all of my tests into one big suite and run it inside Maya, removing the need of command line tool for test discovery.


## Base set of tools

*TODO: don't feel like below is really relevant to the article* 

* `Ubuntu`: Linux is really my platform of choice for development projects. It was a bit tricky to migrate Visual Studio setup of C++ projects and change debugging habbits, but I don't really look back anymore;
* `Eclipse`: some hate it, some love it; I personally have a +10 year relationship with Eclipse going on and it's a great universal IDE, from Python and C++ to writting articles of this site;
* `Git`: not a lot of alternatives for this great version control system;
* `dccautomation`: a great library to remotely run things in Maya. This is the primary way of IDE <-> Maya communication.

## Running Maya in known state 

One of the things I realized at some point that it’s really beneficial to not use default Maya launch profile. When Maya starts crashing for no reason, and you checkout a know stable code, and it’s still does not work as before, one of the things might be that something changed in how Maya is launched, and you want to control that.

![Maya environment - working copy and snapshot](/assets/maya-test/maya-env.jpeg)

In my setup, I keep a snapshot of Maya home folder in version control, and create a fresh copy to actually use when running Maya. The reason to use a snapshot is that I want to version control all files in this folder (to always have a known and version controlled state of maya), but don’t want to spam version control with changes maya does to this folder after every launch.

## Launching Maya
Maya is setup to launch from Eclipse, I use a custom python script for that, which does the following:

* Setup Maya home directory: recreate it from snashot, set up `MAYA_APP_DIR` to point to that location;
* Sets python path to include python project paths; launcher script creates environment variable, which is later parsed by userSetup.py to be picked up by maya.
* Depending on launch mode, runs Maya in “development” or “production” mode. In production mode, Maya is configured to use build-produced module file structure instead of “live” python sources. This is sort of a shortcut of “let’s install what we actually distribute and test if it works”.
* Configures and starts dccautomation server: this listens for any launch requests from IDE. This again gets initiated in userSetup.py of this special Maya home folder I’m using.

Main reasons to run Maya from IDE:

* Version-controlled way to configure maya for development, with all benefits version control provides;
* Maya output can be seen in Eclipse console; as both plugin debug messages and python test output is redirected to maya stdout, there’s a nice feedback right back to IDE as tests are being executed.

## Sample tests

Let's have a look at a few typical tests.

First, there are some slim unit-tests that are barely Maya specific, so I just approach them like normal unit tests, trying to mock whatever I can, and test smallest possible bit.  

This is for a class that is responsible for creating a logical index map for how influences map from right to left; as you can see, it straightforwardly describes how `InfluenceMapping` class should be used: create and configure the instance, add influence metadata, run `calculate()` and resulting "source->destination" map can be found in `.mapping` member.

{% highlight python %}
def testManualOverrides(self):
    mapper = InfluenceMapping()
    mapper.nameMatchRule.setPrefixes('L_','R_')
    mapper.manualOverrides = {3:4, 4:4}
    mapper.sourceInfluences.append(InfluenceInfo(path='L_a', logicalIndex=0))
    mapper.sourceInfluences.append(InfluenceInfo(path='R_a', logicalIndex=1))
    mapper.sourceInfluences.append(InfluenceInfo(path='L_aa', logicalIndex=3))
    mapper.sourceInfluences.append(InfluenceInfo(path='R_aa', logicalIndex=4))
    mapper.sourceInfluences.append(InfluenceInfo(path='a', logicalIndex=44))
    
    mapper.calculate()
    self.assertEqual(mapper.mapping,{0:1, 1:0, 3:4, 4:4, 44:44})
{% endhighlight %}



It is completely viable to write a test like this upfront, without even having a 
Being able to write tests like that requires to have some sort of 

While a trivial tests that are completely disconnected

{% highlight python %}
def testBuildInfluenceMappingEngine(self):
    openMayaFile('simplemirror.ma')
    cmds.select("testMesh")
    self.mll.initLayers()
    self.mll.setCurrentLayer(self.mll.createLayer("test layer"))

    initWindow = self.mirrorTab.execInitMirror()
    initTab = initWindow.content
    
    # prefix parsing        
    initTab.controls.influencePrefixes.setValue('a,b')
    engine = initTab.buildInfluenceMappingEngine()
    self.assertEqual(engine.nameMatchRule.prefixes,('a','b')) 
    
    # drop spaces 
    initTab.controls.influencePrefixes.setValue(' L_, R_ ')
    engine = initTab.buildInfluenceMappingEngine()
    self.assertEqual(engine.nameMatchRule.prefixes,('L_','R_')) 
    
    # ...
{% endhighlight %}

<hr />

### TODO:
* Include links to other TDD articles
	* [Going TDD: first steps](http://www.cesarsaez.me/2014/08/going-tdd.html) by [Cesar Saez](http://www.cesarsaez.me/)
