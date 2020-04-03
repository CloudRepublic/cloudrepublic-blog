---
title: Unit Testing improvements
date: 2020-03-15 09:23:54
tags:
  - Unit Testing
  - Unit Testing improvements
  - xUnit
  - Mocking
  - Unit Test Naming
  - Clean Code
---


Tests are stories we tell the next generation of programmers on a project. If you are to lazy to write them well consider moving to a different profession. This isn't a job for you.

Source: [Roy Osherove - The Art of Unit Testing](https://www.manning.com/books/the-art-of-unit-testing-second-edition)

In a world where we use CI / CD pipelines to automate our releases unit tests are an essential part of a successful software project. Not only do they help us to debug the code we are writing faster but they also help us to validate the code before releasing it to our users.

In this post, we will go over the steps you can take to improve your unit tests and therefore create a better product in the end.

# Naming

Using a good naming convention for a unit test does not necessarily help you to write a better product but it will help your team to identify clearly what unit is tested and what state the unit under test is in.

One of the most know conventions that are currently used is `[MethodUnderTest]_[Scenario]_[ExpectedResult]`. While it is not a bad naming convention it is limited. Writing the naming of the unit test this way leaves out some imported details and might not make it clear to other programmers what the test is trying to test. With naming it is better to tell the story of the test. Let's look at two examples.

In this example, we are looking at a static method that checks on an order if the delivery is valid. Using the convention mentioned above we would get something like this:

`IsDeliveryValid_InvalidDate_ReturnsFalse`

While this is giving us a lot of information about what the unit under test is supposed to do it does not describe the scenario that we want to validate. Let's look at a better example:

`Delivery_with_invalid_date_should_be_considered_invalid`

The above example describes the behavior of the application we are trying to test rather than the code that will be part of the test. This also will help new programmers to understand the application better with limited domain knowledge.

# Test method layout

Another way to make your tests more readable and separate the parts of your tests is by using the `Arrange`, `Act`, and `Assert` pattern or `AAA` for short. using this pattern we split our test into 3 distinct parts.

The `Arrange` section is used to initialize objects and gather data needed for the test or set local variables.

The `Act` section is used to invoke the method that this test is designed for.

The `Assert` section is used to verify that we received the expected result from invoking the method under test.

Using this pattern a test could look like this:

![/images/unit-testing-improvements/code.png](/images/unit-testing-improvements/code.png)

# Simplifying tests and reusing code

If you are writing tests you often have to instantiate objects that are under test and bring them into a specific state to perform the test. Instantiating all these objects in the test method can clutter up your test and lead to a lot of duplicate code.

The builder pattern is here to help. With this pattern, we can bring the objects into a state step by step to meet the requirements of our test. To use the builder pattern we create extension methods that allow us to buildup the object to the desired state. This helps to keep our tests clear, more readable and results in less duplicated code during testing.

In this example, we will be testing the method that checks if an order is ready to be shipped. Since the order is an object that is heavily used in the system we would not want to create it over and over again. Let's have a look at the order object:

![/images/unit-testing-improvements/code%201.png](/images/unit-testing-improvements/code%201.png)

As you can see we have a method called `CanShip`. We should test this method to ensure that for example, we do not ship orders to customers that are not paid yet. We might also want to test if `CanShip` still returns false if the Address is not set yet. We already have two tests to write where we need a new instance of the Order class in a specific state. Creating a new instance of the Order class every time from scratch would not be efficient.

So let's start making use of the builder pattern to buildup the Order class. First, we need a base state that we call `Default` and we create an extension method for it:

![/images/unit-testing-improvements/code%202.png](/images/unit-testing-improvements/code%202.png)

With this extension in place, we can start and write our first unit test for the Order class. The unit test will look as follows:

![/images/unit-testing-improvements/code%203.png](/images/unit-testing-improvements/code%203.png)

As you can see from the picture above we used the extension method to get the order in the default state. What this default state means for you or your team can depend on the object in the test or on how the object is used in the system. In our example default just sets all the fields to a valid value and sets the status to new.

From hereon, we can keep using this method in other tests as well. Now let's imagine we add order-lines to our order class like this:

![/images/unit-testing-improvements/code%204.png](/images/unit-testing-improvements/code%204.png)

We can now create a new extension method for our order that allows us to build a default order with order-lines. The extension method would look like this:

![/images/unit-testing-improvements/code%205.png](/images/unit-testing-improvements/code%205.png)

In the method above I've created the order-line from scratch but let's say we want to test this object in multiple tests we could create the same `Default` extension method as we did for the order class. We can now use this extension method in our next test like this:

![/images/unit-testing-improvements/code%206.png](/images/unit-testing-improvements/code%206.png)

As you can see we can make the test a lot smaller and with less repetitive code. It is like ordering a pizza and saying which toppings you want on the pizza.

# Mocking

With mocking we emulate behavior during our tests that are outside of the scope that we need to test but is needed for the test to run. A good example would be a method the does some validation logic and then gets an order from the database. We would want to test the validation logic but not the database call because this would be more like a regression test.

In the .Net world, there are a lot of Nuget packages that can help you with mocking but one of the more popular ones is `Moq`. This is also the package we are gonna use. While writing the mocking of for example the database call we can use some of the things we learned earlier. In the example below, we are gonna use the builder pattern again to get rid of the mocking code and move it to an extension method.

Lets take the example we described above. The method could look like this:

![/images/unit-testing-improvements/code%207.png](/images/unit-testing-improvements/code%207.png)

We are checking that the `orderId` is not null or empty and then we are calling the `GetOrder` method to get the Order from the database.

If we want to mock this method we need to set up a mock instance of the database class. The setup for this mock will look like this:

![/images/unit-testing-improvements/code%208.png](/images/unit-testing-improvements/code%208.png)

As you can see we are using the Default extension here on the Order class to create a new default Order. Now we could set up the database every time like this in every test we use but that would be tedious to write the same code every time so let's create another extension method:

![/images/unit-testing-improvements/code%209.png](/images/unit-testing-improvements/code%209.png)

With this extension method, we can quickly set up a new database and return a default order whenever we need to during testing. So let's see what a test would look like using this extension method:

![/images/unit-testing-improvements/code%2010.png](/images/unit-testing-improvements/code%2010.png)

In the beginning, it seems like a lot of extra work to create all these extension methods but ones you have a lot of tests that reuse the extension methods they feel like a big time saver.

# Conclusion

As we saw in the scenarios described above we can make some nice improvements to our unit tests to slim them down and make the code more reusable. I would not recommend to rewrite all you tests straight away but ones you need to fix a test or write new ones you can use these tips to make them better.