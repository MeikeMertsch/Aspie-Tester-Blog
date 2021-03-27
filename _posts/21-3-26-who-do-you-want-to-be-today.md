---
layout: post
title: Who Do You Want To Be Today?
published: true
tags: learning randomization privileges learnings automation roles access
---



Does your product have a way to login as an admin? When you test things, or work with it on a daily basis, who do you do it as? An admin?


# Roles, Privileges, and Access

In Plugsurfing we have two restrictive systems to tell what you should or shouldn't see or be able to do in our platform.

The first is your role which determines what actions you are allowed to take. We have almost a dozen roles that your account could have assigned to it. If you are an end-customer you have none. And for all back-office users (we call them system users) you have at least one role. Admins can do just about any action in the system. But only our own staff may have that role. All of our customers have one or more of the other roles that come with their own set of privileges.

The second is our system of organisation access which determines which data you can see and operate on. And that is carefully regulated according to GDPR and contracts between the different players. Again, only members our staff can see everything (and even that is regulated). Anyone else has restricted visibility.

So back to the first question: Who do you want to be today?


# A Realization

At this point I am working for more than four years for Fortum. And most of that time I have been using our internal admins first and foremost to test functionality. Obviously that is the role and access level that allows me to do everything and see everything so it is a good user to check if a feature works. Or is it?

If it works with an admin that can see everything, then the feature is working? No. Not at all. Last year we had at least two occasions where we put out a new feature and, embarrassingly enough, we had to admit no one could use it. Apart from our own staff. Because it was accidentally restricted to admins from our internal organisation.

This wasn't the first time, either. I had the realisation that I needed to test with at least one customer role and a customer organisation already much earlier. First I did it on every feature in both of our test environments (test and stage). Eventually I kept consistently to the system of testing everything with an admin in test and when switching to testing in stage I would avoid admins and our internal organisation unless the feature was explicitly for internal use only or the setup required admin help. On top of that I varied my testing in the second test environment further: I use a different browser and when it comes to testing our app, then I use different screen sizes in the different environments.

In the test automation I had a special setup for this as well. I kept the required privilege settings for the different features and would heavily invest in parameterised tests that would run with the various roles and organisation restrictions. Usually I wrote a test for each action that would run with two roles that were allowed to do an action and two that were not. This I would cross product with the internal organisation that can see all data and a test organisation with restricted access to data. So in total I would run 8 cases for each action as a first test (unless there were fewer than two roles allowed or disallowed for an action).

If a case failed I would see if both organisations would for example fail on the same role or if the access for a test organisation was restricted for all roles.

Recently I came across some tests that were exclusively using admins from the internal organisation for verification of functionality. And it struck me. 


# The Way Forward

This was a pretty useless exercise. What value does it bring if any non-internal feature is tested with an admin? Insignificant. Yes, it tells me if the action can generally be done. But for the customer it has no practical value if an admin can do something if no customer can. We don't make any money off our staff in that way.

This realisation caused me to do two things very differently since.

1. I avoid admins already in the first test environment and not only in stage. I still switch roles and browser and organisation and screen size where possible and applicable between environments. It lets me cover more ground without any more effort. But I cover the customer's ground from the start. I now consider admins as just a tool to get at information or into a state I need for testing. And that means I get an even closer look at how the system looks for a customer. From day one of testing. And even if I am not able to test in both environments, for example when we share the workload, then I can be sure the feature has been seen from something like the customer perspective (well, the product is still no black box for a tester like it would be for a customer).

2. I removed the admins, the logic for choosing suitable allowed and disallowed roles, as well as some more duplication from the complete api test suite. I now test with only customer roles. And I avoid the internal organisation. Now each test runs only once per role (before it was twice: with the internal organisation and with a customer organisation). However, if any test fails, it gets repeated up to twice over to check if it is one of our unfortunate privilege check failures. And we still see if one role in particular has a problem or if it was just some hiccups.

But doesn't that mean that I don't know if everything still works with an admin? Yes. So what? Most developers, internal staff, internal customer support, the product owners, they all use admins because it is simple. And I still do, too. As I said, they are now a source of information and a means to reach state. But no longer a target for testing. If something doesn't work with the admins or the internal organisation, someone of our staff will find out and tell me/us. And if they don't, then we probably don't need it. I don't need to waste my time and effort on testing something that doesn't help customers.

Would you in my shoes?
