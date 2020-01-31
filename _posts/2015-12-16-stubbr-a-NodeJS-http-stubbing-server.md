---
title: Stubbr - a NodeJS HTTP stubbing server
date: 2015-12-16 00:00:00 +00:00
tags:
- NodeJS
- Testing
- REST
layout: post
author: Craig Shipton
---

When it comes to writing integration tests, constructing a test sandbox that abstracts away every HTTP service dependency lets us concentrate on testing the interactions within our own app in isolation.  I've found mock http servers to be an invaluable tool for achieving this; this post is to discuss the mock http server that we created to serve this need in a greenfield [AngularJS](https://angularjs.org/) app integration tested with the [NodeJS](https://nodejs.org/)-based [Protractor](http://www.protractortest.org/) test framework.


## Initial thoughts

Having done a fair share of Java integration tests in the past using Tom Akehurst's [Wiremock](http://wiremock.org/), it seemed like the immediately obvious choice:

* Well supported and frequently updated? Certainly.
* Feature rich? Definitely.
* Easy to use? Yep!
* Can require it via [NPM](https://npmjs.com) and use it in an Angular test?  Less easy.

*Ah...*

It was at this point that we had to decide if we were going to just mock Angular's [$http](https://docs.angularjs.org/api/ng/service/$http) service with the [$httpBackend](https://docs.angularjs.org/api/ngMock/service/$httpBackend) mechanism, write a NodeJS wrapper around Wiremock, use a 3rd party Javascript mock server library, or simply just write our own.  In the interests of keeping the environment of our deployable as similar to the live running app as possible, we made the decision not to mock HTTP responses via $httpBackend - pushing us towards a mock server solution.  When considering the remaining options we had to have flexibility and the agility to add functionality as progress on the greenfield project continued.  Using a 3rd party JS solution would mean feature pull requests with no certainty of continuous updates and writing a JS wrapper around Wiremock would still leave us at the mercy of its feature roadmap.

Taking the above into consideration, and the desire to see just how quick it could be done, we wrote a quick Node mock/stubbing server app that would serve as a proof of concept - Stubbr.

## Structure

From the very start the entirety of Stubbr was written as a Node module, with [ExpressJS](http://expressjs.com/en/index.html) serving as the web framework that enabled us to be up and running with a first iteration within a tiny amount of time.

In terms of design, we wanted to be able to serve a stub for a combination of different HTTP request properties - the combinations of which being an *expectation*.

# The first iteration

On the first iteration of the app, our requirement was to simply be able to serve a stub for an expectation made up of only URLs and HTTP methods.

A typical example could be represented by this logical structure:

|           |GET      |POST    |
|:---------:|:-------:|:------:|
| /a/url    | stub1   | stub2  |
| /b/url    | N/A     | stub3  |

where stub1, stub2 and stub3 are expected responses with their own response statuses and payloads.  

Our actual data model is then a Javascript object looking like so:

```json
{
  "GET /a/url": {
    body: stub1.response.body, // stub1's json payload
    status: stub1.response.status, // stub1's integer status code
    count: 0 // how many times this stub has been requested
  },
  "POST /a/url": {
    body: stub2.response.body,
    status: stub2.response.status,
    count: 0
  },
  "POST /b/url": {
      body: stub3.response.body,
      status: stub3.response.status,
      count: 0
    }
}
```

The number of scenarios we could we could express with these changes was already fairly comprehensive on a macro level; happy path scenarios could be configured by returning 200-range HTTP responses on all method + URL combinations or you could sprinkle 400 or 500 error responses as necessary.

Of course, there's more to an everyday HTTP request than just a HTTP method and a URL; it became quickly apparent that what we had was not expressive enough for anything non-trivial! Stubbr *had* to support request/response headers and request bodies if it were to remain relevant.

# Current iteration

On the current (and second) iteration, we needed to be able to store different stubs for combinations of methods, URLs, request headers and request payloads.  It was on the realisation of this requirement that the two-dimensional map structure we currently had was inherently limiting; we figured an internal tree type structure for storing expectations would be incomparably more flexible and thus more appropriate.  In this tree structure, each node would represent a property of an expectation with zero to many child nodes joined by links and a maximum of one attached stub.  These links between nodes form a sequence of properties, from the root to the leaf node, which forms the expectation.

You can see how we would store the above stubs in this format:

![Expectation Tree]({{ site.github.url }}/images/{{page.date | date: "%F"}}/ExpectationTreeSmall.svg)

### Expectation path

A key part in interfacing with the tree is the use of an *expectation path* which forms a single "branch" of the tree, starting with the compulsory URL node, of variable length.  When submitting requests to Stubbr (whether stub creation requests, or requesting stubbed data), the expectation path is the internal representation of the request.

Examples of expectation paths are covered in the following section.

#### Deepest common node

Every operation that we do with the tree requires finding *the deepest common node* (DCN from here) between itself and a path, then manipulating the tree from that node.  We locate the DCN by optimistically comparing the tree with a path for equivalency on a node-by-node basis, starting from the top and working our way down the tree.  

For tree and path nodes to be treated as a match they must:

* Have the same type, e.g. URL, method, request body.
* Have equivalent values - according to a equivalency function for that particular node type.

A successful match means we can disregard any sibling nodes and check the child nodes, whereas an unsuccessful match moves to the next available sibling.  The search terminates when there remain no nodes that could possibly match our request, returning the DCN between the current state of the tree and the given path.

### Building the expectation tree to add stubs

Adding a stub is as simple as finding the DCN in the expectation tree, then adding the stub to it if it's a leaf node, or appending the rest of the path from its DCN onto the tree.  The root of the tree always remains unchanged!

#### Adding a stub to a new leaf node

For a new expectation path with a stub called 'stub4':

![Expectation Path]({{ site.github.url }}/images/{{page.date | date: "%F"}}/ExpectationPathStub4.svg)

The URL and method nodes matched, making the POST node the DCN.  Our tree now has the extra two nodes from the path appended, with the stub attached to the request body node:

![Expectation Tree]({{ site.github.url }}/images/{{page.date | date: "%F"}}/ExpectationTreeStub1234.svg)

#### Adding a stub to an existing node

For a new expectation path with a stub called 'stub5':

![Expectation Path]({{ site.github.url }}/images/{{page.date | date: "%F"}}/ExpectationPathStub5.svg)

The header node created for stub4 is the DCN and the stub is attached to it:

![Expectation Tree]({{ site.github.url }}/images/{{page.date | date: "%F"}}/ExpectationTreeStub12345.svg)


### Navigating the expectation tree to find stubs

Finding a stub is the exact same process as adding a new stub. After finding the DCN in the expectation tree (if there is one), we just need to return its stub (again, if it has one).  In the previous examples, a request that satisfies stub4 could also satisfy stub5 - although this wouldn't happen as we always use the most specific stub possible.

## Finally!

We've only done the bare minimum to suit our needs, and what we've created is serving our needs on an almost daily basis.  That being said, we recognise that there's much more which can be done to improve Stubbr and a few examples of where we plan to take it are:

* Exposing a built-in visual representation of the expectation tree stored within a running instance, served from a new URL.
* Generally adding more possible properties to an expectation - cookies, images etc.
* Supporting more payloads than just JSON.

and more!
