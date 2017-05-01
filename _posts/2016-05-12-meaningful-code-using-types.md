---
layout: post
title:  "Meaningful Code Using Types"
date:   2016-05-12
author: Michael Hatch
excerpt: Crafting meaningful code is vital to the long term success of a project. Here I'll discuss how we can use types to help us to acheive this goal. I'll also go through a few helpful IntelliJ refactors that help introduce our types.
---

A few months ago I travelled to London with a group of colleagues for the JAX London conference. We saw many talks on a wide range of topics from DevOps to Java 8.

In one of the most memorable presentations, Samir Talwar explored how to make the most of a language's type system. The full blog can be read [here](http://talks.samirtalwar.com/use-your-type-system.html). Samir discussed how the use of types, rather than primitives, can be really useful to make your code easier to understand and maintain. 

Here I want to discuss recent work in the development of Shippr, an internal application created at Auto Trader. To put it concisely, Shippr...

> Manages the building of environments and the process of releasing changes to those environments.

I’ll draw on some of the points raised by Samir and show some refactors that can be performed. The refactor shortcuts here are all IntelliJ based; sorry Eclipse users.

First lets start with some code that creates a Fleet (a collection of servers) on a third party cloud management platform. We pass in a unique Fleet Identifier and a human readable Fleet Name and get back a Cloud Fleet Identifier which we use later to refer the our newly created Fleet. A farm is the third party name for a collection of servers and essentially equates to a Fleet.

```java
public int createFleet(String fleetIdentifier, String fleetName) {
    String farmName = fleetName + "_" + fleetIdentifier;

    AsyncHttpClient.BoundRequestBuilder createFarmRequest = new AsyncHttpClient().prepareGet("http://scalr-api-end-point");
    createFarmRequest.addQueryParam("action", "create-farm");
    createFarmRequest.addQueryParam("farm-name", farmName);
    ListenableFuture<Response> execute = createFarmRequest.execute();

    String createFarmResponseBody = null;
    try {
        createFarmResponseBody = execute.get().getResponseBody();
    } catch (ExecutionException e) {
        throw new RequestException("Unsuccessful create farm request", e);
    }
    int farmIdentifier = Integer.parseInt(parseText(createFarmResponseBody).selectSingleNode("//FarmCreateResponse/FarmID").getText());

    AsyncHttpClient.BoundRequestBuilder launchFarmRequest = new AsyncHttpClient().prepareGet("http://scalr-api-end-point");
    launchFarmRequest.addQueryParam("action", "launch-farm");
    launchFarmRequest.addQueryParam("farm-identifier", "" + farmIdentifier);
    launchFarmRequest.execute();

    return farmIdentifier;
}
```

We can see in this code that we are currently simply passing around primitives and relying on the variable name to give them context. This is obviously better than having nonsensical names sucha as ```a```, ```b``` and ```c``` but it is still not ideal.

This brings us to our first improvement...

Readability
-----------

Let's take a look at the signature of this method, as seen by a client of the Class to which it belongs. Using our IDE we would see something like:

```java
int createFleet(String fleetIdentifier, String fleetName)
```

The parameters...
========

Understanding the parameters in this context is not too difficult. However, if we were calling it from a precompiled library, we may not have the luxury of the parameter names being preserved.

The return value...
======

Here we have an ```int```. As our method is called ```createFleet```, we can't really deduce much about either the value that will be returned or its use. Put simply, we aren’t giving the reader the context they need to properly understand our code. Instead all we can deduce from the signature is that we need to pass in two Strings; we are telling it to create a Fleet and we get back an ```int```.

The refactor...
======

An easy way to create a wrapping object is to use the 'Parameter Object' refactor action. From within the method on Windows press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>A</kbd> or on a Mac <kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>A</kbd> to open the Action Search pane. Once here find the 'Parameter Object...' option. In the new window you will be given the chance to name and place your new object.

![Extract Parameter Object]({{ site.github.url }}/images/{{page.date | date: "%F"}}/fleet-identifier-parameter-object.PNG)

This can be repeated for each of the parameters. The method signature which remains is much more pleasing:

```java
int createFleet(FleetIdentifier fleetIdentifier, FleetName fleetName)
```

We are still returning an ```int``` from the method, giving the client no clue as what to expect. But luckily, another IntelliJ refactor can help with this. Again we need to open the Action Search pane (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>A</kbd> or <kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>A</kbd>) this time using the 'Wrap Method Return Value...' to enter an appropriate name for your new return type. Ours is a ```CloudFleetIdentifier```.  Our signature now looks like this:

```java
CloudFleetIdentifier createFleet(FleetIdentifier fleetIdentifier, FleetName fleetName)
```

This is much easier to read and allows us to easily build a model of the code in our heads.

Readability is vital to the process of development. It facilitates the transfer of context between the original developer and the next person working with the code. Any way to speed up this transfer of context is clearly beneficial.

Static Analysability
--------------------

Another advantage of using types is that it helps your IDE to more easily statically analyse the code while unlocking a host of refactors. This is extremely useful in finding usages, renaming, etc.

Typed parameters help the IDE when performing refactors. If we take our previous refactor,

```java
int createFleet(String fleetIdentifier, String fleetName)
```

and for some reason we wish to add another ```String``` as the first parameter, the IDE finds it hard to guess the new order of the parameters and consequently provides a list of confusing options.

![List of confusing refactor options]({{ site.github.url }}/images/{{page.date | date: "%F"}}/introduce-parameter-options.png)

As Samir suggests, we can spend less time worrying about what we might need to change and more time reading and writing new code. Essentially, we can use the IDE features to find the exact uses of a ```DeploymentIdentifier``` rather finding all ```String```s in the code and somehow filtering them by their name to find the ones that actually represent a Deployment Identifier.

Encapsulation
-------------

When starting a project we often have virtually no idea what the final code will look like. This is especially true when dealing with ever evolving requirements. Our new types can help with this. When we create a new type we also create a **Behaviour Attractor**. This is to say it makes sense for some system behaviour to belong to our new class. A simple example in our code above is the point at which we are constructing the ```farmName``` String. We created wrapper object for the parameter objects earlier that are used to create the ```farmName```.

```java
String farmName = fleetName.name() + "_" + fleetIdentifier.identifier();
```

This is some behaviour about how a ```farmName``` is constructed. If we needed to create multiple of these across the codebase, we will end up recreating this code. We see that we have some common code that needs to go somewhere. But where? Once we have created a ```FarmName``` object we now have that place.

The refactor...
===============

To do this let's first refactor the ```String farmName``` into ```FarmName farmName```. There is, unfortunately, no fast and easy way to refactor a ```String``` to a custom type. But I suggest simply changing the code as follows:

```java
FarmName farmName = new FarmName(fleetName.name() + "_" + fleetIdentifier.identifier());
```

This allows IntelliJ to fix the compilation errors for us by using the quick fix or 'Show Intention Actions' pop-up. To do this on both Windows and Mac press <kbd>Alt</kbd>+<kbd>Enter</kbd> and select the 'Create class' option. If we do this with our text cursor on the constructor side of the statement, it will create the class and the constuctor taking the parameter ```String```; this would not happen if we performed this action on the left hand side. The code is now compiling. We now have our behaviour attractor. Now let’s extract a method to create the Farm Name from the ```fleetName``` and ```fleetIdentifier```. To do this on Windows press <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>M</kbd> on a Mac <kbd>Cmd</kbd>+<kbd>Alt</kbd>+<kbd>M</kbd> to open the 'Extract Method' pane as below.

![Extract Farm Name]({{ site.github.url }}/images/{{page.date | date: "%F"}}/extract-farm-name-method.PNG)

As can be seen this isn’t too complicated. We can then use this newly created method as a factory method on the ```FarmName``` class. Let's make it static by selecting the 'Declare static' check box. We then have a method that looks like:

```java
private static FarmName farmName(FleetIdentifier fleetIdentifier, FleetName fleetName) {
    return new FarmName(fleetName.name() + "_" + fleetIdentifier.identifier());
}
```

We can then move the method by moving the text cursor to the method and pressing <kbd>F6</kbd>. Find the newly created ```FarmName``` class and move.

I tend to statically import these factory methods as their names and signatures make them self explanatory. Leaving them fully qualified can also make lines too long and reduce ease of reading.

The only thing left to do it change the visibility of the constructor that takes the ```String``` to private. We do not intend for people to use this constuctor as the format of a Farm Name will always be created from a ```FleetIdentifier``` and a ```FleetName```.

We have now moved some behaviour that would have been spread around the code, encapsulating the behaviour and making it easier to change in the future.

We can use a similar technique to encapsulate our requests into request objects. We can see that we have two different requests, one to create a farm and one to launch the farm. Where we create the farm, we also want to extract the Farm Identifier that comes back in the response.

There is quite a lot of logic around performing a request to create a farm, in fact over half the method.

```java
AsyncHttpClient.BoundRequestBuilder createFarmRequest = new AsyncHttpClient().prepareGet("http://scalr-api-end-point");
createFarmRequest.addQueryParam("action", "create-farm");
createFarmRequest.addQueryParam("farm-name", farmName.name());
ListenableFuture<Response> execute = createFarmRequest.execute();

String createFarmResponseBody = null;
try {
    createFarmResponseBody = execute.get().getResponseBody();
} catch (ExecutionException e) {
    throw new RequestException("Unsuccessful create farm request", e);
}
int farmIdentifier = Integer.parseInt(parseText(createFarmResponseBody).selectSingleNode("//FarmCreateResponse/FarmID").getText());
```

This is an obvious candidate for its own type. Let's extract a ```CreateFarmRequest``` object. Our new ```CreateFarmRequest``` object will have a single field holding the built ```AsyncHttpClient.BoundRequestBuilder``` and an execute method which will return us the identifier for the newly created farm.

To begin let's again extract a method by selecting the first three lines of code and using the keyboard shortcut from before (<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>M</kbd> or <kbd>Cmd</kbd>+<kbd>Alt</kbd>+<kbd>M</kbd>). This will create a method that returns an ```AsyncHttpClient.BoundRequestBuilder```.

To create our new object we will again make use of the 'Wrap Method Return Value...' from the Action Pane (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>A</kbd> or <kbd>Cmd</kbd>+<kbd>Shift</kbd>+<kbd>A</kbd>). After these steps we will have a method that looks like below:

```java
private static CreateFarmRequest createFarmRequest(FarmName farmName) {
    AsyncHttpClient.BoundRequestBuilder createFarmRequest = new AsyncHttpClient().prepareGet("http://scalr-api-end-point");
    createFarmRequest.addQueryParam("action", "create-farm");
    createFarmRequest.addQueryParam("farm-name", farmName.name());
    return new CreateFarmRequest(createFarmRequest);
}
```

Now we are in a position to extract the code that executes the request and extracts the Farm identifier from the response. The next step is to capture the result of calling ```createFarmRequest(farmName)``` in a variable, rather than leaving it in-line. This will allow the next method we extract to take this variable as its parameter.  Next we extract the code that gets the ```AsyncHttpClient.BoundRequestBuilder``` from the newly created variable from above, executes it and extracts the Farm identifier. This block of code forms our execute method. As an instance method on the ```CreateFarmRequest``` we will not make it ```static```.

After this method is extracted we are able to move it onto the ```CreateFarmRequest``` with <kbd>F6</kbd>. Then we move the factory method we created earlier.

If we perform a similar refactor and create a ```LaunchFarmRequest``` our code will be as below:

```java
public CloudFleetIdentifier createFleet(FleetIdentifier fleetIdentifier, FleetName fleetName) {
    FarmName farmName = farmName(fleetIdentifier, fleetName);
    int farmIdentifier = createFarmRequest(farmName).execute();
    launchFarmRequest(farmIdentifier).execute();
    return new CloudFleetIdentifier(farmIdentifier);
}
```

We have managed to encapsulate the logic to make a HTTP request to our third party meaning that our method above has no knowledge of the ```AsyncHttpClient```.

Flexibility
-----------

In order to keep up with the changing nature of our requirements, successful code relies on its flexibility. We cannot predict what problems we may encounter or what new requirements will arise. Using our type system helps us to support this.

Lets look at our code again:

```java
public CloudFleetIdentifier createFleet(FleetIdentifier fleetIdentifier, FleetName fleetName) {
    FarmName farmName = farmName(fleetIdentifier, fleetName);
    int farmIdentifier = createFarmRequest(farmName).execute();
    launchFarmRequest(farmIdentifier).execute();
    return new CloudFleetIdentifier(farmIdentifier);
}
```

Now that we have introduced request objects, we are in a position to change the inner workings of the request objects. When we create wrapper objects we establish a contract with the interacting classes. Therefore as long as we do not wish to alter this contract, refactors are easy. Here our ```CreateFarmRequest``` contract is that the object will do something to create a farm and inform us of its FarmIdentifier.

In this example we are using the ```AsyncHttpClient```  to build and execute our requests. If we wish to change this library because it has, say, a major security issue or there is a new shinier way to make HTTP requests, we can simply alter our Request classes and not worry about what may be affected.

To sum up
---------

After various refactors the code we end up with is easier to understand and refactor and much more flexible. This stands us in good stead for the next time somebody needs to read the code and build a model of it in their head&mdash;ideally, the exact same model we had when writing the code!