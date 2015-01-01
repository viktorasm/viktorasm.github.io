---
layout: post
title:  "Automated testing and Maya plugin development"
date:   2015-01-01 20:00:00
categories: TDD Maya
---

Writing another article on automated testing might seem silly at first. TDD is exhaustively discussed already, and there's plenty of good material about it already, why another?

I thought that instead of writing a tutorial, or a guide of some sort, I just want to share my story, and though process of transitioning into TDD setup in [ngskintools.com](http://www.ngskintools.com) project.

I'll probably be skipping quite a few technical details on how this or that is implemented. I will assume that the reader who's interested in such a topic would have some necessary expertise to fill in the gaps.

## The problem

My previous development workflow was a real mess:

* Ad-hock shell buttons in Maya to launch tests, reload plugin binary or python code, and various other utilities;
* C++ plugin output went to Maya standard output, Python output went to Script Editor history; tracking "what's going on" during execution meant a lot of hit and miss;
* Various "useful" code snippets in script editor to run in Maya, for various things I would be working on at the moment;
* Not a lot would be automated in first place, so workflow would generally mean: code, build, alt-tab (or worse, relaunch maya if it crashed from previous run), load test scene, perform some actions in UI, observe result, repeat;
* Putting project aside and getting back to it after a couple of months would require quite a lot of fiddling around to get things to build again, run stuff in debug mode, figure out how to test this or that;
* In the even of having to rebuild my development environment, preparing Maya for such workflow would require quite some manual fiddling around, getting all required paths and buttons setup again;  


## The goal
The whole thing needed ... structure. The best automated test setup is one-click setup. You checkout source code on another computer, say "run tests", and it just works. This is how things work out of the box for a lot of project setups out there, but to get to this point for Maya plugin project will require quite some effort.

The ice needed to be broken in some uncomfortable areas, like "how do I reliably run tests from IDE inside Maya and get the success/error feedback back", "how to write UI tests", "how do I switch between running a single test and whole suite", and so forth.

Another big goal was to require as little alt-tabbing as possible; having to switch between windows is a real productivity killer for me. Using Eclipse for both C++ and Python side of plugin, I'm already down to one tool to write and build code. What about tests? It would be nice if I could:

* Run Maya from Eclipse (more on that later);
* Run tests from Eclipse;
* Route all outputs to single console in IDE;

Ideally this would allow me to just sit in Eclipse and perform the whole development cycle in one tool.

## The setup in action


*TODO: describe current setup in action*



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

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
