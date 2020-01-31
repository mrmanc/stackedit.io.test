---
title: Exception Handling Conundrums
date: 2017-09-05 00:00:00 +01:00
tags:
- Java
- Code Craft
layout: post
author: Chloe Hodgson
---

Our squad recently came across an unhelpful error message* in response from one of our APIs. After some digging around in the code, we discovered that the message was due to an exception that had not been handled appropriately and instead had bubbled up. This prompted us to sit down as a team and agree on some best practices around how we will deal with exceptions. This post will cover what we agreed on, in particular how to handle crossing knowledge boundaries, using examples from our codebase.

\* The error message looked like this:

```
{
  "code": 500,
  "message": "There was an error processing your request. It has been logged (ID ea2935735c8281e2)."
}
```


#### What Should I do With the Stack Trace?
It's important to always preserve the stack trace for diagnostic purposes, and whilst there are a few ways that stack traces can be lost, these are very easy to prevent. One way a stack trace can be lost is if you catch and rethrow a new exception, without passing the original exception as an argument. Rethrowing an exception somewhere else in the code will also create a different stack trace, causing any context you have collected to be lost. 

In order to prevent losing the stack trace, you should always provide the original exception as an argument to the new exception's constructor. The same goes for logging—if you catch and then log, you should add the exception to the end of your log message. The following is a good example of where the stack trace has been preserved:

```java
} catch (FleetStateUpdateFailedException e) {
    LOG.error("Something went wrong whilst attempting to update the state of fleet '{}'.",
    fleet.identifier(), e);
}
```

#### Should I Catch `Exception`?
It is almost always a bad idea to catch `Exception`. It’s better to be as specific as you can when catching exceptions so that you can handle them appropriately. For example, if your code throws an `IllegalArgumentException`, you would most likely want to handle this differently to if it was a `NullPointerException`. 

Additionally, catching `Exception` means that the method you are calling could later change its signature to throw a checked exception, and you would never know about it. The intention of throwing checked exceptions is to ensure that the client handles that exception specifically. By catching all possible exceptions, you’re forcing the client to handle them all in the same way.

#### When to Handle and When to Throw?
You should only catch an exception if you want to either handle it, or wrap it and rethrow. When handling exceptions, as a general rule you should log the exception. When rethrowing, you should wrap it in a more appropriate exception and include extra context.

In terms of logging, exceptions should only ever be logged once—you shouldn’t see the same error being logged multiple times. The log message should provide information on what the application was trying to do when the exception was thrown, and what potentially went wrong:

```java
} catch (FileNotFoundException e) {
    LOG.error("The file {} was not found.", file.getPath(), e);
}
```

#### What Should I Throw?
It’s always better to throw your own context-specific implementation of Exception. For example, RuntimeException isn’t specific enough—by creating your own implementation you can ensure that the exception name is relevant and has more meaning—this is covered in more detail in the next section of this post, but in the meantime here is an example of where we have done this in our codebase:

```java
} catch (DuplicateKeyException e) {
    throw new DuplicateVirtualIpException(virtualIp.address());
}
```

Here, we are catching an exception which is thrown when attempting to insert something that already exists into a Mongo collection. As you can see, we have wrapped this in our own `DuplicateVirtualIpException` which is a more relevant name to us, as we now know that this has to do with the Virtual IP collection specifically. If we had left this exception as a `DuplicateKeyException`, it could have related to any of our Mongo collections. This is particularly useful at the point of diagnosis as we are now able to narrow down what went wrong to a specific part of our codebase. 

#### But What About Knowledge Boundaries?
A knowledge boundary is the place in which code from one module of an application crosses over to another module. This often happens when dealing with exceptions. For example, an exception thrown in one module may be caught in another. 

Generally, it is wrong for exceptions to cross these knowledge boundaries without being dealt with. It's good practice that no exception should ever cross a knowledge boundary without extra context being added. The way we decided to handle this in our codebase is to create our own exceptions, with relevant names, and place these in a location that can be accessed by both modules. This way, we can throw the exception in one location and catch it in another. An example of where we have done this in our codebase is below:

```java
} catch (RetryException maxRetriesReached) {
    throw new DeploymentWatchMaxRetriesReachedException(deploymentUrl, desiredState.state(), maxRetriesReached);
}
``` 

As you can see, extra context has been added to the exception message which would be useful at the point of diagnosis. We've also preserved the stack trace of the original exception by adding it to the constructor of the new exception. 

It is also important to note that the name of these exceptions must make sense in the context of the knowledge boundary they are in. In other words, the exception should be named in the language of the client code and not the server code. 

An example of this in our codebase is our `TrafficManagerOperationFailedException`. This could have been named `F5OperationFailedException`, since F5 is the Traffic Manager that we use. This would be fine inside the F5 module where it is thrown, however since this exception crosses a knowledge boundary and is caught in a separate module, this name would not make sense. This is because the other modules only know that there is a Traffic Manager, they don’t know about F5.  

#### Checked or Unchecked?
If an exception is crossing a knowledge boundary and you want to force the client to wrap it in something more appropriate, then it would be okay to make the exception checked. 

In general, however, our default choice when creating an exception is to make it unchecked. This means that we do not have to declare exceptions everywhere in our code (in method signatures etc.). It always allows us to bubble up exceptions that cannot be handled correctly by the client. 

