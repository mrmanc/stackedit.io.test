---
layout: post
title:  "Using AssertJ"
author: Nick Baker & Paul Doran
---

When writing tests we aim to use Descriptive and Meaningful Phrases [(DAMP)](http://blog.jayfields.com/2006/05/dry-code-damp-dsls.html). This also reduces the time needed to understand the intent of the test. Making assertions read like the sentences we see every day is one way to achieve this. [AssertJ](http://joel-costigliola.github.io/assertj/) provides a comprehensive set of fluent assertions for Java.

For example, the Junit statement:

```java
assertEquals(expected, actual)
```

in AssertJ becomes:

```java
assertThat(actual).isEqualTo(expected)
```

The assertion now reads as a sentence.

We will now highlight some AssertJ features through the medium of a story about pizzas.

# An everyday scenario
Alice loves pizza. She loves it so much that she has agreed to help her friend Bob make a website for his pizza shop.
As a responsible and forward-thinking developer she adopts a test driven approach. Requirements from Bob will inform the design of tests.

# Chaining Assertions
Alice decides to start by asking Bob about the pizzas themselves. "What toppings do we need on a Pugliese pizza?" she asks.
Bob responds, but due to his vast pizza knowledge and detailed explanations, Alice doesn't have enough time to write down all the requirements in separate assertions!

Thankfully, the fluent pattern followed by AssertJ allows chaining of assertions.

```java
assertThat(pugliese.getToppings())
        .hasSize(3)
        .contains(MOZZARELLA, TOMATO, ONION)
        .doesNotContain(PROSCIUTTO_DI_PARMA);
```

This allows the succinct capture of the test's intent, allowing Alice to write down everything Bob said.

# Collection Assertions
Now that the pizzas themselves have requirements, Alice decides to ask Bob about the orders he receives. When she realises that an order can contain many pizzas, her heart sinks. Testing the complete contents of a collection can be a chore. In this case one the fields, toppings, is a set itself making testing even more of a burden. Alice realises that she can use AssertJ to extract multiple values from a collection and assert upon them.

```java
assertThat(order.getPizzas()).hasSize(4).extracting("base", "toppings")
        .contains(
            tuple(ITALIAN, newHashSet(MOZZARELLA, TOMATO)),
            tuple(ITALIAN, newHashSet(MOZZARELLA, TOMATO, PROSCIUTTO_DI_PARMA)),
            tuple(ITALIAN, newHashSet(MOZZARELLA, TOMATO, ONION)),
            tuple(DEEP_PAN, newHashSet(MOZZARELLA, TOMATO, ONION, PROSCIUTTO_DI_PARMA))
        );         
```

Extracting only the fields she wants to test provides clarity to the reader on what is being checked in the collection. She clarifies the test's intent by referencing only the subset of the fields she wishes to verify.  Reinvigorated, Alice carries on.

# Custom Error Messages
Whilst Alice is happy to help Bob build the website, she knows she can't continue to maintain it forever. She needs to make sure that the output of every test is unambiguous for whoever works on the site.
Instead of the minimalist error given by JUnit, the AssertJ output is already clear, returning something like this:
```expected:<[4]> but was:<[3]>```.
This clarity means spending less time wondering what is being tested, so work can quickly begin on fixing the broken code.

If the output requires even more detail then the errors message can be customised.

```java
assertThat(order.getPizzas()).as("Order should have size 4").hasSize(4);
```

which returns

```
[Order should have size 4] size:<4> but was:<3>
```

Satisfied that she won't confuse future generations of developers, Alice powers on.

# Asserting Exceptions
“What about when things go wrong?” Alice wonders. She knows that even the best code will run into problems at some point, so she plans ahead. AssertJ allows the testing of Exceptions like all other code artifacts. For example

```java
@Test
public void shouldThrowExceptionWhenOrderHasNoPizzas(){
  Order order = a.order().withPizzas(newArrayList()).build();

  Throwable actual = catchThrowable(order::getPizzas);

  assertThat(actual).isNotNull()
          .isInstanceOf(EmptyOrderException.class)
          .hasMessage("No pizzas in order.");
}
```

This enables Alice to follow her beloved arrange/act/assert pattern when testing Exceptions.

# Generating Assertions
All this work has made Alice tired, and the huge amount of pizza consumed whilst working hasn't helped either. She really doesn't feel like writing tests for all of her domain models today. Once again, AssertJ has her back.

AssertJ has the ability to [generate assertions](http://joel-costigliola.github.io/assertj/assertj-assertions-generator.html) for  domain objects. For example, for a Customer class with fields "name" and "address", generating assertions would give:

```java
assertThat(customer).hasName("Joe").hasAddress("123 Another Street")
```

This enables concise expression of the assertions' intent using the domain terminology. As well as a command line tool to do this, AssertJ also provides plugins for Maven and Gradle.

# Summary

AssertJ improves the tests' readability, output and maintainability. It is well documented and IDE friendly, so along with code completion it is easy to learn and use. The open source project is active, so if you find a missing feature, add it and contribute back to the community. More pizza related examples of AssertJ features can be found  [here](https://github.com/dorzey/assertj_pizza_example).
