---
layout: post
title: Randomisation in Test automation
published: true
tags: learning test_automation learnings automation junit randomization property-based_testing property_based_testing
---

Recently I realised that I am doing something that is not particularly helpful. Let me explain what it was and how I dealt with it.


## How it started

In 2010 I listened to a [conference talk](https://www.xpdays.de/twiki/bin/view/XPDays2010/UnitTestsAlsSpezifikation) about property based testing and generation of random test data with tools like [QuickCheck](https://pholser.github.io/junit-quickcheck/site/1.0/) and co. A few years later I blogged about it [on my old private blog](https://agiletester.webnode.com/news/tweaking-automated-checks-part-three/). The gist is that, instead of testing with the same data all the time, you generate test data on the fly during the test with set properties. On a unit test level it allows you to use [shrinking](https://pholser.github.io/junit-quickcheck/site/1.0/usage/shrinking.html)[^1] to find out more about a failing test.

I really liked that idea for several reasons:

* It's very boring to always test with "John Doe" living on "Main Street". And test data gets unrecognisable since it always looks the same. You wouldn't notice if "The Earl of Grey" always gets created twice because of code duplication if all you have is your Earl Grey.
* To have the generation of test data decoupled from the tests can also make the tests more readable since [they don't get cluttered with data that is not relevant for a test](https://confluence.fortum.com/display/LA005994/2021/01/29/My+journey+with+Model+data).
* You automatically test anything with more than just one very specific input. On unit test level with very fast tests, shrinking will allow for rerunning the test to learn more and you could generate data with and without special characters and so on.

I've been working with randomised data ever since (whenever it seemed to make sense for the project).


## Project background: Two problems in Plugsurfing 

We have about 1100 test cases in the api automation of the shared platform currently. They run at least once a day completely, but subsets also on check-ins of some other repositories and of course they run on every change of the tests.

Since a long time (years, really) we are aware of a problem with the system. Every now and then, calls to the service that handles access control for users fail at random. You can imagine that this is a very fundamental part of the system since every request to the api has to go through this part of the system to check what the calling user may do and what data they are allowed to access. And it sounds like you would immediately fix this problem. We actually spent a good bunch of days (weeks?) of two developers a while ago to try and fix the issue; unsuccessfully. We think the problem are sudden losses of a db connection. Eventually we stopped that effort since, in practise, this happens rarely for every singular user and a simple reload usually solves the issue. However, the automation faces this problem on an almost regular basis since it makes so many calls in rapid succession. According to a recent count, a third of all failed test runs in the months between October and December 2020[^2] can be attributed to this problem. We live with it.

A second problem for the automation is the use of Elastic Search (ES) and lambdas in our system. With queues, possibly cold lambdas and the inner workings of ES, objects are not queryable immediately anymore. Instead you should have a short window of a few seconds delay until an object is put into a place from where you can query it. The problem is that this short amount of time sometimes is not enough. Especially when the automation with some parallelised tests puts a strain on the system. Then it happens here and there that an object is not created after 30, 60, or even 90 seconds. And almost another third of all test failures in the timeframe mentioned above are related to this second issue. Sometimes moments, sometimes minutes after the test, the object gets created and is queryable. No problem. Just late. Well, and then again, the time for such things should be shorter. And again tests fail randomly and point out that an expectation about the system is not met.


## What I did (wrong)

With that said, I eventually realised that I made some mistakes with my efforts to randomise the automation.

I forgot to take under consideration that we work with a system that has such fundamental problems and can fail randomly. There's a reason why people undertake big efforts to get rid of flaky tests. In this case it is not a flaky test though. The tests point out very real problems of the system under test. And almost any test can fail because of these issues[^3]. With that in mind you realise immediately that it's very likely that we attribute a "random" failure of a test to either of these issues instead of to a finding of the randomisation where the system has hiccups with certain specific test data points. Shrinking or no. And that renders the effort for randomisation for broader testing completely useless.

A second thought about a possible mistake I made was to not use a randomisation tool like QuickCheck. Just RandomUtils and RandomStringUtils. Those tools do more than just throw in some random data. Again, shrinking comes to mind. However, and this is the reason why I am unsure if it really was a mistake: The automation that I am involved in is not on unit test level. On the contrary. The e2e tests are long runners. It costs some effort to keep running times of the main suite below 10 minutes without reusing the very same objects over and over. So when shrinking comes into play, tests have to be repeatable often. But some tests, under specific circumstances, can run several seconds and more. Especially when they are dependent on Elastic search and Kafka. So I am hesitant to hook them up to investigation tools that might extend run times of tests drastically on failures which happen relatively often.

## What I changed recently

I didn't want to remove the randomisation altogether. The effect of distinguishable data is useful in too many cases. But I wanted more control of my randomisation. I wanted data that looked realistic. It demoes better, too:

Before and After:
![Before the new Randomization]({{ site.baseurl }}/images/Screenshot%202021-02-22%20at%2020.02.07.png) ![After the new Randomization]({{ site.baseurl }}/images/Screenshot%202021-02-22%20at%2020.02.58.png)

So I decided to invest a little time into a customised Random class. With a few dozens random adjectives and random nouns, as well as random first and last names and some UUID magic (and a few other things with just a minimum of logic) my test data started to look almost like real data. I still "tag" the data for the automation with a specific prefix and the time of testing to make sure it's recognisable as data from the automation and for the clean up to find the data that can be removed[^4].

![Excerpt of my first spike of a Random class]({{ site.baseurl }}/images/Screenshot%202021-02-22%20at%2021.05.20.png)


## Conclusion

This way I now have test data that ...

* ... I fully control. Unless I explicitly test for special characters or something, I can expect the Random class to always give me valid data.
* ... is not completely boring. It looks almost like production data and it's easier to identify and remember a specific data point: not the "11th John Doe" but the "first Patrick on the page" between all sorts of other names.
* ... should demo nicely. When we show test data it's not just gibberish but people can actually identify an address or a company name.
* ... lends itself well to decoupling. If you write "A user with Random.address...", it's pretty apparent that the exact values of the address are not important for the case.

* ... does not cover as much ground anymore. But that is practically irrelevant since we never really used that extra ground. Unless an error message is very explicit about what validation the tests might have upset. And that is far from implemented everywhere.

I doubt we would have attributed a failure correctly as the randomisation identifying a problem. With huge json blobs of test data, the underlying db connection issue, and things like lambdas and queues that might fill up and need emptying before the system can deal with it all, I suspect it is far more likely that we would attribute every such failure to the typical system issues.

So now the automation might succeed slightly more often. Let's hope. In any case it got better in other areas.



---

[^1]: 'When junit-quickcheck disproves a property for a given set of values, it will attempt to find “smaller” sets of values that also disprove the property, and will report the smallest such set, leading to an easier bug investigation for the developer.' -- junit-quickcheck documentation.

[^2]: Excluding repeatedly failing runs for the same exact reason like a bug that wasn't fixed by the next run.

[^3]: We have tests that don't run against the main apis and therefore don't hit that critical functionality around the db. Others have nothing to do with ES or lambdas.

[^4]: The clean up of this automation is done asynchronously, usually two days after object creation. It allows for some investigation times (the objects are not just gone after a test) and the running times are by far smaller if they don't include the tear down.



