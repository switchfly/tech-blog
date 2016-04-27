---
layout: post
title: Speeding Up Builds
author: Ian Haken
author-image: ihaken.jpg
summary: Utilizing features of Maven and Surefire to parallelize and speed up builds.

---

# Speeding Up Builds

Testing is good. Testing saves developers time, catching bugs sooner rather than later. Testing saves money, catching bugs before they hit production. Testing shortens feature releases, allowing developers to change large bodies of code with confidence that problems will be caught by tests with [high code coverage](https://www.atlassian.com/software/clover/overview).

But testing is also bad. If your code base is over a million lines and sports over 10,000 unit tests, your one-line change in a globally used library could effect a _lot_ of code. And re-running every test to make sure your change didn't break anything can take a long, long, looooong time. And those 5, 10, or even 15 minutes you "spend waiting":http://xkcd.com/303/ for those tests to run add up when multiplied out by dozens of developers running hundreds of builds a day.

With that in mind, let's explore the features of [Maven 3](https://maven.apache.org/index.html) -- Switchfly's chosen tool for project and build management -- which can help speed up our builds by taking advantage of the ever-increasing capabilities of our hardware.


## Parallel Module Builds

When Maven 3.0 was released in October 2010, it added support for [parallel builds](https://cwiki.apache.org/confluence/display/MAVEN/Parallel+builds+in+Maven+3), analyzing the dependency graph of your modules and building them in parallel threads where able. This is a very cool feature; all it takes is a flag on the command line to specify how many threads to use (e.g. `@mvn -T 4 clean install@`) and it +Just Works+(tm). However, this feature has some limitations.
* The extent to which your build can be parallelized depends on the dependency graph of your modules. If you have a highly connected or linear dependency graph, this feature may gain you nothing at all.
* The "build" of each module includes all its tests. So if you have a large common library with thousands of tests which is used by every other module, then all the tests for this module will be run _before_ any other module's build is started. Ideally those tests could be run in parallel with the building and testing of the other modules.

So the parallel builds in Maven 3 are great for some applications, but aren't going to get you a whole lot of gains if you've got a highly-connected dependency graph, or if the goal is to speed up the testing phase of your build's lifecycle.


## Parallel Test Execution in Surefire

If you're using Maven, then you're almost certainly using the [Surefire plugin](https://maven.apache.org/surefire/maven-surefire-plugin/) to run tests. And if you're in a situation similar to the above where you've got a single module with _lots_ of tests, then there's also lots of gains to be gotten by executing those tests in parallel instead of sequentially. And fortunately Surefire also has a collection of features to support [parallel test execution](https://maven.apache.org/surefire/maven-surefire-plugin/examples/fork-options-and-parallel-execution.html).

Surefire has supported the [@parallel@](https://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html#parallel) option for TestNG tests since its 2.2 release way back in 2006. Using it (along with its sibling parameter "@threadCount@":https://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html#threadCount) you can tell Surefire to spawn several threads and execute the tests in parallel. This is a fantastic and easy solution, so long as:
1. You're using TestNG.
2. Your tests are thread-safe. Remember that this option runs tests in separate threads on the _same JVM_.

The latter is the particularly prickly point, since it inhibits tests from being able to successfully test code which need to use or modify static classes. If your code base has static classes to perform tasks such as settings management or dependency injection, then instructing Surefire to run in parallel with threads is a surefire recipe for failed builds. And even if your code is thread-safe, are you sure that every third-party library you depend on is thread-safe as well? Furthermore, even if all of your code and all the third-party code it depends on is thread-safe, if your tests read and write to a database then you're still going to run into problems as your tests simultaneously step on each other's data.

So as before, this feature can provide massive speed-up for some scenarios, but has some significant drawbacks. In particular, unless you're lucky and have only the right kind of tests, you're likely to end up with test failures.


## Parallel Test Execution with Forking

Fortunately Surefire's features for parallelized tests don't stop there. In version 2.14, released in March 2013, the Surefire team added the [@forkCount@](https://maven.apache.org/surefire/maven-surefire-plugin/test-mojo.html#forkCount) option, deprecating their old @forkMode@ option in the process. `@forkCount@` works similarly to `@threadCount@` above, except it spawns separate JVMs instead of using threads. This way the tests that are running in parallel are completely isolated from one another, and can modify static classes as needed without effecting each other. Furthermore, Surefire can inject a fork id number into the system properties, so that tests can identify in which fork they are running. Surefire demonstrates in their "documentation":https://maven.apache.org/surefire/maven-surefire-plugin/examples/fork-options-and-parallel-execution.html how this can be used to set a separate database schema for each forked JVM. This solves the secondary problem mentioned above of having parallel tests using the same database. And even better, Surefire is smart enough be aware of Maven's multi-module parallel threading (from the first section, i.e. @mvn -T 4 clean install@), and inject a fork id number so that there are no overlaps. So you can run `@mvn -T 2 -DforkCount=4 clean install@`, and wind up with 8 forked JVMs, all accessing their own independent database schema.

The only disadvantage with this method is that it's necessary to do some additional work in order to create and manage the additional databases, and to setup your POM and test code to be aware of and use the `@${surefire.forkNumber}@` placeholder. However, this feature opens the doors to full parallelization of thread-unsafe tests with no significant code changes or refactors. With just a few options passed to Maven, you can set every core of your CPU to blaze through your tests and cut your build time in half (if not more)!


## Conclusions

With only a small degree of effort, it's possible to use the features of Maven and Surefire to parallelize pieces of the build lifecycle and cut build times down significantly. The features recently made available in Surefire allow for complete test isolation so that tests which previously could only be run sequentially can also be parallelized -- all with little to no changes to the code itself. The result is builds that take half the time, if not less!

And now that your builds are taking half as long, you can stop spending all day browsing blogs and get back to coding!
