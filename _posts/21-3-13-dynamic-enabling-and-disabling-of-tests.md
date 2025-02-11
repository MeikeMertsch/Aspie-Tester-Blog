---
layout: post
title: Dynamic Enabling and Disabling of Tests
published: true
tags: learning test_automation ignore disable disabled learnings automation junit jenkins gradle
---

Or: How we treat tests that point to a problem which won't get fixed immediately.

How do you deal with failing tests when they uncover a bug that will **not** be fixed right away? If test runs are red over more than a handful of builds, the information value of a failing test suite falls drastically. No one will pay attention anymore, since "they always have been red" or "it's just that problem we know".

In Plugsurfing we disable tests that point to known issues unless they can and will be fixed very fast. The difference between a green and red build is bigger than the difference between two failing tests with slightly different counts of failing tests. It makes your ears prick up and should cause an investigation of what the reason for the failing tests is.

Over the last few years, we changed the concrete actions to disable tests quite a bit.


## Step 1 - Disabling tests

At first we went the typical way. Test frameworks come with an idiomatic way to disable a test. For JUnit that's annotations. Let's take an example:

{% highlight java %}
@Disabled
@Test
void testSomeVeryImportantFuncationality() {
    // test code
} 
{% endhighlight %}

It causes the test to be disabled fully. As long as a test has such an annotation, it simply won't run. For starters that was enough. We did add one piece of information immediately. In JUnit 5 you can add a reason to a disabling annotation. So we used it to mark tests with the respective ticket number. It makes the tests easily searchable and if we find a test which is disabled we can find out more about the reasons by checking the ticket. Usually we also put a note in the ticket that there is a connected test.

{% highlight java %}
@Disabled("LA-1234")
@Test
void testSomeVeryImportantFuncationality() {
    // test code
} 
{% endhighlight %}

The annoying part of this approach is that we have to remove the annotation to check if the test is eventually successful. Anyone working with the breaking reasons cannot simply run something on Jenkins but has to go through the process of cloning the repository and change the code to try it out.


## Step 2 - Tagging tests

Since a while we went one step further: Instead of using the hard "Disabled" annotation, we switched to using simple [Tags](https://www.baeldung.com/junit-filtering-tests). For example

{% highlight java %}
@Tag("LA-1234") 
@Test 
void testSomeVeryImportantFuncationality() { 
	// test code 
} 
{% endhighlight %}

Now the tests were not disabled in JUnit, but we could tell the build tool Gradle to filter out tests with specific tags. And to make things easier, we stored the configuration in our Organization Configuration Service. That service got a new configuration type Test Automation Configuration and now stored the information of what to filter out.

The upside of this is that with a little addition to our Jenkinsfile, even if a test is tagged and marked as disabled, anyone can use Jenkins to run that test and ignore the disabling. Jenkins got new parameters includedTags and excludedTags to be able to overwrite the dynamically loaded config. If a developer had a fix deployed, anyone could run the test(s) for the fix on Jenkins to try out if it worked.

![Jenkins parameters]({{ site.baseurl }}/images/Screenshot%202021-03-14%20at%2012.54.55.png)

Once the tags were in place, we could switch the tests on and off at any odd time. On jenkins we had some output that would tell the reader which tags were currently ignored.

The downside is that we still had to commit, peer review and push code for the tags to be present and after the fix was in place, a second round of those steps had to be taken to get rid of the tag later.

A second downside is that by using Gradle, JUnit never got word that the excluded tests even existed. That meant that the number of skipped tests was always underreported. Filtered tests did not show up in the numbers.

And a third downside was that removing the Disabled annotation meant that even the failing tests would now run locally, because Jenkins was taking care of the config and locally there was no Jenkins involved, so JUnit wouldn't know what to exclude.


## Step 3 - Fully dynamic enabling and disabling of tests

To make all of the pains with the previous steps go away, we proceeded to engage in a fair amount of Vodoo. Ok. No vodoo. But we used some neat little tricks to solve the issues at hand.

A first revelation was that Gradle doesn't only support disabling by tags but also by other things. One of them is by class and method name. So we finally had a way to not being dependent on any annotations at all. After a smaller refactoring and giving Jenkins, Gradle and the automation config new instructions, we could finally enable and disable tests without any code changes at all. Any annotations were now unnecessary.

One down, two to go.

Instead of letting Gradle filter the tests out, we switched to having JUnit take care of that part. Luckily, we already had a common Base Test for all tests to take care of some basic logging and holding a minimum of constants and helper utils that were needed in practically all tests.

In the base [BeforeEach](https://www.baeldung.com/junit-5#1-beforeall-and-beforeeach) method we took advantage of a testing hook that JUnit offers: [TestInfo](https://junit.org/junit5/docs/current/api/org.junit.jupiter.api/org/junit/jupiter/api/TestInfo.html). This reveals class name and method name. Exactly what we needed to check against the configuration. Since JUnit5 offers [Assumptions](https://junit.org/junit5/docs/current/api/org.junit.jupiter.api/org/junit/jupiter/api/Assumptions.html), we could now check before any test if the test should run and if not, set it to ignored on the fly. That solved the problem of the undercount of skipped tests. And brings back the possibility to see why a case is ignored, since Assumptions also allow to define a message, just like the Disabled annotation.

And it also set the stage to solve the third problem: Local runs. Now that JUnit was responsible for disabling tests, we simply added a step in the class that is handling the environment variables to fetch the same configuration from the Organization Configuration Service. A property switch can be used to enable or disable that configuration.

With all of that we reached our goal to fully dynamically be able to enable or disable tests from any and all Jenkins runs as well as local execution. No more hectic Friday afternoon PRs to disable some tests for the weekend. One or two simple API calls are all that is needed now to keep the focus on unknown issues.

But wait. Aren't there any downsides?

Of course there are. At least one. This method is using [reflection](https://www.baeldung.com/java-reflection) to find the tests. So we have to be mindful when changing test names and test class names. But so far nothing really stands in our way.

## The future - A conclusion

I am personally very happy with the solution and what it brings us. Fewer PRs need to be taken care of to just flip a switch. It also means that the disabled tests are now very explicit and we can even tell the automation to exactly run all of the disabled tests to see if something has been fixed without anyone noticing or re-enabling the test.

There is one more thing I want to do: Sometimes only one or two sets of parameters in [parameterized tests](https://www.baeldung.com/parameterized-tests-junit-5) need to be disabled. For the moment that means commenting out some parameter sets or setting up more or less readable Assumptions like so:

{% highlight java %}
@MethodSource(DIFFERENT_USERS) 
@ParameterizedTest 
void testSomeVeryImportantFuncationality(User user) { 
	assumeFalse(MANUFACTURER.equals(user.role()), "LA-1234"); 
	// test code 
} 
{% endhighlight %}

Since we have the TestInfo object, we should be able to use it to disable the exact set instead of disabling the whole test or needing to use a different approach. If that works, then we can do much more than we ever could.

And while implementing all of this, I learned a lot and found some solutions for other problems left and right. It was worth the journey but I am glad we arrived at something satisfying now.
