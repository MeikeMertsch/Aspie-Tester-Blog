---
layout: post
title: My journey with Model data
published: true
tags: learning test_automation learnings automation junit readability information
---

## A quest for clarity and readability

We all know the situation when we want to send a simple call to the backend, be it via Swagger, automation code, or via tools like Postman and Insomnia. And we need to send an object with a bunch of parameters. If we get lucky it's just a few parameters that we are very familiar with, like the fields of a billing address on a customer. But it could be a more complicated object like an organisation, that might have some more or less deeply nested sub-objects like contact information, billing data and delivery addresses at once.

When describing a test, most exact values are unimportant for a huge percentage of test cases. Usually we only need valid data for most or even all values. And if we provide all data in the test description, it easily distracts from the relevant information. Let's take an example:

{% highlight cucumber %}
Given a customer with mail address tcore.fortum+something@gmail.com and password test 
 
When I add an address John, Doe, Grand Plaza, 321, 12345, Somewhere fun
 
Then adding should have succeeded
and the address is John, Doe, Grand Plaza, 321, 12345, Somewhere fun
{% endhighlight %}

Of course it's easy to say, "but we will likely test with different variants of this and have the actual data in a table with the different examples". Ok, sure, this can be a little more readable. But if you look at this test you will still not really see what exactly changed and what is important unless you boil it really far down to something along the lines of 

{% highlight cucumber %}
Given a random customer
 
When I add a random address
 
Then I should get a success
and the address is correct.
{% endhighlight %}

All the clutter is now gone. And you can see without any effort what the test is about, even if it doesn't have a title. 

On Swagger we need to hope for the patience and mercy of the developers to actually have half way valid test data in the examples. And there is no real way to tell from a json blob what the important part of the data was. On Insomnia/Postman we can save some samples for later use but have the same problem. The real possibilities come in play when we start looking at code for automation.


## A long ride
Over the last almost four years I have been writing a lot of code. As everyone else I started small and it took me a long time to get something half way useful together. Over the years I have learned a lot through code reviews by the programmers, through reading, scouting the web, and through conferences. I think I re-wrote the code several times by now. And with every new feature that comes with completely new models I make the same journey on a smaller scale again. I start with writing all fields explicitly first. Then I generalise more and more over time. Let's check another address example and let's assume I have some random customer already created somehow (note that this is pseudo code  and not correct Kotlin, I try to keep it as simple as possible) :

```java
// given
val customer = someAlreadyCreatedCustomer
val payload = AddressOf("John", "Doe", "Grand Plaza", "321", "12345", "Somewhere fun")
 
// when
val result = api.send(customer, payload, addressEndpoint)
 
// then
assertThat(statusCode(result)).isEqualTo(200)
assertThat(parse(result)).isEqualTo(payload)
```

Usually I pull the exact values out of a test very early. Both variable names and method names give us the possibility of giving something meaning. If I tell you one variable name is randomName and another is nameWithSpaces, you probably know immediately what the relevance of those objects is. If the name doesn't explicitly say otherwise, I always assume I have a valid object.

Once I started using some objects more often and manipulating different parts of the object, a simple method didn't suffice anymore. Like only changing the billing address of an organisation in one test but just the contact details in another. That's when I started using the Builder Pattern and pulled bigger objects into separate classes where I filled them with valid default values and could them then manipulate:

```java
// given
val customer = someAlreadyCreatedCustomer
val payloadWithSpecialStreet = DefaultAddress().withStreet("Grand Plaza").build()
 
// when
val result = api.send(customer, payloadWithSpecialStreet, addressEndpoint)
 
// then
assertThat(statusCode(result)).isEqualTo(200)
assertThat(parse(result)).isEqualTo(payload)
```
```java
class DefaultAddress() {
 
    firstName = "John"
    lastName = "Doe"
    street = "Highway"
    street_number = "321"
    zip = "12345"
    city = "somewhere fun"
 
    fun withStreet(newStreet: String): DefaultAddress {
        this.street = newStreet
        return this
    }
 
    fun build() {
        return AddressOf(firstName, lastName, street, street_number, zip, city)
    }
}
```

## Seeing the castle on the hill
That worked well for a while. However, I noticed some patterns of things that I needed over and over again. And I really wanted to be able to write something like

```
address.sendToApi(user)
```

Why? I didn't want anyone to need to care about how something is exactly sent to the server. When a user presses a button, they don't know the endpoints used. Similarly I thought those model classes were perfect for holding the information of how their data reaches the server.

So a lot of my classes with objects that were filled with default values got a new parent class that they would inherit similar behaviour from. It meant fewer lines of duplicated code. And all classes derived from that parent needed to implement a method that contains the endpoint to which to send the payload, a method to build the request, and a method of how to deserialise the object in the response. Outside of the model classes I wouldn't necessarily need to deal with that anymore.

(While writing this, I realise that I actually forgot to use this in many places. I need to change that!)

```java
class DefaultAddress() extends Model<Address, AddressResponse>{
 
    firstName = "John"
    lastName = "Doe"
    street = "Highway"
    street_number = "321"
    zip = "12345"
    city = "somewhere fun"
 
    fun withStreet(newStreet: String): DefaultAddress {
        this.street = newStreet
        return this
    }
 
    inherited fun build() {
        return AddressOf(firstName, lastName, street, street_number, zip, city)
    }
 
    inherited fun getEndpoint() : String {
        return "/v1/some/important/address/endpoint"
    }
 
    inherited fun retrieveObject(response: Response): AddressResponse {
        return wayOfDeserialisingThe(response)
}
```
```java
class Model() <RequestClass, ResponseClass>{
 
    abstract getEndpoint(): String
    abstract retrieveObject(response: Response): ResponseClass
    abstract build(): RequestClass
 
    api = someObjectThatAllowsCallingTheApi
 
    fun sendToApi(user: user): Response {
        api.send(user, build(), getEndpoint())
    }  
}
```
```java
// given
val customer = someAlreadyCreatedCustomer
 
// when
val result = DefaultAddress().withStreet("Grand Plaza").sendToApi(customer)
 
// then
assertThat(statusCode(result)).isEqualTo(200)
assertThat(parse(result)).isEqualTo(payload)
```

Yes, the net amount of code is now bigger. But by this state I usually have a ton more than one single test that need all sorts of different configurations to be sent to the server. And the code in the most prominent place, the actual test is more descriptive of the relevant parts and most often shorter, too.


## Reaching the treasury

Since we have more than one api that we go against in our project, and the different apis have their own special rules, I started only now to generalise even more. Some of those default filling objects need to make PUT requests, some do POSTs, some PATCH. And so far I was focused mostly on the POST requesters. Since we have now started to use Kotlin more and more and the builder pattern is much less verbose in Kotlin than in Java (to be honest, what is NOT more verbose in Java? It's the Finnish of programming languages from my perspective (wink) ), I started rewriting many of those classes in Kotlin and am taking that chance to also further unify the different stages of the coding journey by refactoring them. 

On the other side I see a bunch of very useful classes that make it very effortless to get much used objects filled with default values and contain the logic around how to get them properly to the server so it's not scattered all over the code. I'm always happy when I have to reuse one of them and find how simple it is to get a basic but valid object that I don't have to worry about. And when something needs to change it is usually contained in the area around that very object, instead of each test where I needed to add yet another parameter or change an endpoint.
