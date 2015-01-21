---
layout: post
title:  "Automated testing and Maya plugin development"
date:   2015-01-01 20:00:00
categories: TDD Maya
comments: true
---

Writing another article on automated testing might seem silly at first. [TDD](http://martinfowler.com/bliki/TestDrivenDevelopment.html) is [exhaustively](http://martinfowler.com/articles/is-tdd-dead/) [discussed](http://www.cesarsaez.me/2014/08/going-tdd.html) already, and there's plenty of good material about it already, why another?

I thought that instead of writing a tutorial, or a guide of some sort, I just want to share my story, and thought process of transitioning to TDD setup in [ngskintools.com](http://www.ngskintools.com) project.

I'll probably be skipping quite a few technical details on how this or that is implemented. I will assume that the reader who's interested in such a topic would have some necessary expertise to fill in the gaps.

## The problem

During the first couple of years or so, ngSkinTools development workflow was nothing short of a mess, and could be summarized as:
 
<p class="graf--pullquote">I have some tests, but some things are easier to test by hand. I’m kind-of managing it this way.</p>

There would be ad-hock shell buttons in Maya to launch tests, reload plugin binary or python code, and various other utilities. Various "useful" code snippets would be kept in script editor to run inside Maya, for various things I would be working on at the moment, from adjusting python path, to setting up a test scene or launching a particular sequence of commands while I'm debugging something.

Not a lot would be automated, so coding/testing workflow would generally mean: code, build, alt-tab (or worse, relaunch maya if it crashed from previous run), load test scene, perform some actions in UI, observe result, repeat.

C++ plugin output went to Maya standard output, Python output went to Script Editor history; tracking "what's going on" during execution meant a lot of hit and miss.

Putting project aside and getting back to it after a couple of months would require quite a lot of fiddling around to get things to build again, run stuff in debug mode, figure out how to execute tests. In the event of having to rebuild my development environment, preparing Maya for such workflow would require remembering all this special configuration, getting required paths and buttons setup again.  

I know, the whole situation does sound really amature. I guess the reason I ended up in this state was mainly discouraging thoughts like:

*"It's not really clear how you can run tests in Maya"*

*"I don't quite know how I will automate UI tests"*

*"I don't have IDE/tools support for such type of testing"*

*"I already write tests though... just some things are easier to test by hand"*

*"I'm kind-of managing it this way"*

## The goal
The whole thing needed ... structure. A few things are obvious:

* Setup the project to enable *comfortable* test-driven development: write tests for new features, reproduce bugs as tests, etc.; adding, writing and executing tests should be *simple* enough not to be discouraging; 
* Eliminate the need of "useful snippets" and disposable code: wrap utility checks into tests, automate or rethink the need to have such workarounds in first place;
* Reduce the number of actions needed to rerun tests to a minimum. No manual managing of reloading plugin binaries or python code; all prerequisites should be handled by test runner automatically. 
* Remove the need to alt-tab to Maya for test execution: code, run tests and inspect results within IDE, just like for other type of development projects;
* Unify debug outputs: ideally, single log for C++, Python, test outputs, so that the timeline of events is clearer; 


But most importantly of all:

<p class="graf--pullquote">adding, writting and executing tests should be simple enough not to be discouraging.</p>

In other types of projects I'm working on (mostly web-related Java, Python, Node.js, etc), there's a strong support for this. You write a test and it's discovered and added to the test suite automatically. Your tests execute as part of a build.  Major frameworks make it easy to test code written against it. IDE's let you run the whole suite, or just the test you're currently working on at the moment. Supporting comfortable TDD is among primary goals of any decent platform out there.

Couldn't say that about Maya. 

From C++ API side, you can't mock much; the way you have to use API binds you to it. A good example would be `MPxCommand.doIt(const MArgList& args)`; you can't test the method because there's no way to instantiate `MArgList`. Except for cases where you can completely separate the algorithm from Maya API, you'll need to have Maya running, either in it's normal form or as standalone library. Even for something as simple as using a data structure with an MString in it. 

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

Let's have a closer look at a few different aspects of how I approach automated tests in Maya.

### Not really unit tests

I'm not much of a unit test evangelist. I don't care about running my code purely against mocks, or that my whole test suite executes in a second. This especially true for projects like Maya plugin, where you really can't easily mock the rich ecosystem your code executes within.

<p class="graf--pullquote">As long as tests reliably produce consistent behavior, and finish in a reasonable amount of time, it's fine by me.</p>

The main goal of a test for me is to touch as much functionality and to assert it against the expected behavior. 

My typical ngSkinTools test would look like this:

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

Not sure how self-explanatory it is for someone not familiar with my code, or ngSkinTools plugin but I am:

* opening a test scene (versioned along the rest of the code);
* performing necessary scene setup - initializing skinning layers, adding default layer;
* Then actual test: open UI window, set values into controls, then assert that those values correctly configure the internal engine.

No mocks, no placeholders. Pretty much an actual operation a user would perform, just automated. We hit quite a few places of the code here: calls to C++ "backend", UI construction code, construction of `InfluenceMapping`.


Of course, whenever possible, I try and have the more traditional unit-tests when possible. Here the internal behavior of the same `InfluenceMapping` being tested:

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

It's a test of an engine that is responsible for discovering relationships between right and left influences in a skin cluster. As you can see, it's designed to run as a standalone Python object, not relying on Maya API, and as such, I can just test it separately. Just showing that whenever possible, a "proper" test is even easier to setup.

### Running Maya in known state 

One of the things I realized at some point that it’s really beneficial to have a special Maya profile (that folder with `Maya.env`, `prefs`, `presets`, `scripts`) dedicated just for test execution. It would be a completely separate profile from the default one, and most importantly - recreated each time the Maya is started. This ensures that environment your tests execute in is the same each time, with same settings, same plugins enabled, etc. It's mostly to battle strange cases where one day for no reason your whole suite starts crashing, even the past builds marked as stable.

<p class="graf--pullquote">You want to eliminate as much of unknowns as possible.</p>


In my setup, I keep a snapshot of Maya home folder in version control along with other project files, and create a fresh copy to actually use when running Maya:

<div style="text-align: center">
<img src="/assets/maya-test/maya-env.jpeg" alt="Maya environment - working copy and snapshot" title="Maya environment - working copy and snapshot" />
</div>

### Launching Maya
Maya is setup to launch from Eclipse, I use a custom Python script for that, which does the following:

* Setup Maya home directory: recreate it from snashot, set up `MAYA_APP_DIR` to point to that location;
* Sets python path to include python project paths; launcher script creates environment variable, which is later parsed by userSetup.py to be picked up by maya.
* Depending on launch mode, runs Maya in “development” or “production” mode. In production mode, Maya is configured to use Maya module that is produced by production build, including both binary and python sources. This is sort of a shortcut to action of “let’s install what we actually distribute to users and see if it works”.
* Configures and starts dccautomation server: this listens for any launch requests from IDE. This again gets initiated in userSetup.py of this special Maya home folder I’m using.

The benefits of actually running Maya from IDE were discussed above.

### Getting all tests together

Creating a test suite in python from all tests available in your project is a little bit tricky, and best used from command line. There's also things like [Nose](http://pythontesting.net/framework/nose/nose-introduction/), but I decided not to use it either. I did not need all the magic Nose provides, and instead, I wanted another kind of  magic. Not sure if it explains things well enough:)

In the root of tests package, I have a `testsAll.py`, which imports all the submodules, looking for test classes, and for each found test class, decorates test methods to print a separator. It looks like this:
 
{% highlight python %}

packageRoot = os.path.dirname(os.path.dirname(__file__))

def isTestsClass(c):
    '''
    it's a test class if it inherits test case and has at least one test*() method
    '''
    if not issubclass(c, unittest.TestCase):
        return False
    
    for method,_ in inspect.getmembers(c, inspect.ismethod):
        if method.startswith("test"):
            return True
    
    return False;

def decorateTestMethods(c):
    
    for methodName,method in list(inspect.getmembers(c, inspect.ismethod))[:]:
        if not methodName.startswith("test"):
            continue

        def decorate(method):
            m = method
            def decorated(*args,**kwargs):
                splitter = "-- TEST: "+m.im_class.__name__+"."+m.__name__
                splitter += '-'*(80-len(splitter))+"\n"
                sys.__stdout__.write(splitter)        
                return m(*args,**kwargs);
            return decorated
        
        setattr(c, methodName, decorate(method))
        
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

Having this `testsAll` module means that I can use `unittest.TestLoader().loadTestsFromModule` to programmatically load all of my tests into one big suite. The output would look something like this:

<pre>
<code>mapping |joint4|joint5|joint6 to |joint4|joint5|joint6|joint7|joint8
initializing vertex transfer
creating layers
-- TEST: InitTransferWindowTest.testOpen----------------------------------------
checking skin cluster availability
reading vertices
-- TEST: InitTransferWindowTest.testTransferBuildInfluenceMappingEngine---------
checking skin cluster availability
reading vertices
-- TEST: LayerDataTest.testInfluenceList----------------------------------------
-- TEST: LoggingTest.testCreateLogger-------------------------------------------
[ngSkinTools INFO 23:05:55] something's up
-- TEST: LoggingTest.testEnabledFor---------------------------------------------
-- TEST: LoggingTest.testLogMethods---------------------------------------------
[ngSkinTools INFO 23:05:55] info
[ngSkinTools DEBUG 23:05:55] debug
[ngSkinTools WARNING 23:05:55] warning
[ngSkinTools ERROR 23:05:55] error
-- TEST: MainWindowTest.testOpenClose-------------------------------------------
-- TEST: MainWindowTest.testOpenWithInvalidOptions------------------------------
-- TEST: MeshDataExporterTest.testExportData------------------------------------
-- TEST: MllInterfaceTest.testAccess--------------------------------------------
opening maya file..
init layers..
get mask..
get layers..
done.</code>
</pre>

## Summary

Setting up automated tests on Maya is no small time investment. However, once in place, it's really rewarding. It's much easier to keep the pace at which the project is evolving and make sure you're not breaking already implemented things. You can refactor confidently. Your tests even serve as documentation of how your production code can/should be used.

It's also worth noting that the most painful thing is add tests retrospectively for already existing features, so if you've got this new project going and still not sure if you need automated tests for it - you probably do!

