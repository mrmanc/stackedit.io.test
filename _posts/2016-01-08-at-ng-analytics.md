---
layout: post
title: Configurable custom dimensions for AngularJS apps
category: development
tags: AngularJS, Google Analytics, Open Source
author: Thomas Inman
date:   2016-01-08
---

As part of the work on one of our new AngularJS apps we had to look at how we were going to integrate Google Analytics page and event tracking. One of the key points to cover was that it had to be configurable by a range of users across the business. Often config like this would be held in a spreadsheet and converted by hand by the development team into code. Keen to avoid this we tried to come up with a JSON based config that would be both human readable and directly used by the analytics module. The config ended up being split into three main parts; describing custom dimensions, pages, and global events.

We developed a module for use with AngularJS apps that would take in this config in order to define what payloads to send when page and event tracking. As well as being able to set custom dimension values in the config alone the module provides a data layer at its core which the parent app can push values to. This means we can have some custom dimension values that are static across our entire app or given page, and others that are dynamically set by user interactions.

[We have published the module as open source under the MIT license on GitHub](https://github.com/autotraderuk/at-ng-analytics)

Below is a rundown of how we use `at-ng-analytics` along with some discussion on its design.

Configuration
-------------

#### Custom Dimensions

Here's an example of some custom dimensions config.

```javascript
[
  {
    "id" : 1,
    "name" : "Dimension 1",
    "value" : "value1"              // static value
  },
  {
    "id" : 2,
    "name" : "Dimension 2",
    "dataLayerVar" : "dimensionVar" // value from data layer
  }
]
```

The id field relates to the dimension recorded in GA, So `"id" : 1` will be sent as `dimension1`.

#### Pages

The pages config contains the tracking details for the page itself and any events local to that page.

```javascript
[
  {
    "name": "Page 1",
    "state": "pg1",
    "customDimensions": ["Dimension 1", "Dimension 2"]
  },
  {
    "name": "Page 2",
    "state": "pg2",
    "customDimensions": ["Dimension 1", "Dimension 2"],
    "defaultDataLayerValues": {
     "dimensionVar": "dimensionValue"
    },
    "events": [
      {
        "name": "Event 1",
        "category": "standard-link",
        "label": "event1"                // static value
      },
      {
        "name": "Event 2",
        "category": "standard-link",
        "label": "event2",
        "labelDataLayerVar": "eventVar"  // value from data layer
      },
      {
        "name": "Event 3",
        "category": "standard-link",
        "label": "event3",
        "customDimensions": ["Dimension 3"]
      }
    ]
  }
]
```

`at-ng-analytics` is built on top of [AngularUI](https://angular-ui.github.io/)'s [UI Router](http://angular-ui.github.io/ui-router/site/#/api/ui.router) so will detect state (page) changes and send page tracking events if they are configured, no extra code required. The `state` field matches the name given to UI Router's [$stateProvider](http://angular-ui.github.io/ui-router/site/#/api/ui.router.state.$stateProvider) when configuring states.

Event tracking includes custom dimensions configured for the page as well as any configured just for the event e.g. Event 3 will be sent with custom dimensions 1, 2 and 3. Event tracking does require a little extra code, to register click handlers, which is covered later in this post. The label value for event tracking can be obtained dynamically from the data layer by adding a `labelDataLayerVar` field.

We realised that quite often a custom dimension value will be static on a per page basis, so we provide a way of setting this using default values. They are optionally provided for each page using `defaultDataLayerValues`. If the data layer doesn't hold a value for the variable when an event or page is tracked then the default will be used instead.

#### Events

Some events aren't limited to a single page (e.g. menu bar click events) so are configured separately.

```javascript
[
  {
    "name": "Event 1",
    "category": "navigation",
    "label": "event1"                 // static value
  },
  {
    "name": "Event 2",
    "category": "navigation",
    "label": "event2",
    "labelDataLayerVar": "eventVar"   // value from data layer
  },
  {
    "name": "Event 3",
    "category": "standard-link",
    "label": "event3",
    "customDimensions": ["Dimension 3"]
  }
]
```

 When these events are tracked they will include custom dimensions for the current page as well as any configured just for the event.

Event Tracking
--------------

 To enable event tracking on html elements we put together a directive that is applied as an attribute.

```html
<button at-ng-event-tracking="labelValue">Click Me</button>
```

The value given to the directive matches the value of the `label` field for the event in the config.

Often we need to set a data layer variable before the event tracking is performed. The `at-ng-event-tracking-data` attribute takes an object that gets merged to the data layer before event tracking is triggered. Importantly this is a resolved object so we can take variables from our scope/controller and pass them right through.

```html
<button at-ng-event-tracking="labelValue"
  at-ng-event-tracking-data="{\'dimensionVar\':scopeValue}">Click Me</button>
```

Using the Data Layer
--------------------

Lots of custom dimensions for a page tend to vary e.g. the id value of the page we are loading, `4985934` in `example.com/order/4985934` or `example.com/order?id=4985934`. So we use the `AnalyticsDataLayerService` to set these in the init method of page controllers as these get resolved before page tracking is triggered.

```html
AnalyticsDataLayerService.setVar('orderId', '4985934');
```

Future Work
-----------

The current version of the module is reasonably featured and suits its purpose for our app. There's definitely more features that need working on though. For example at the moment only clicks are able to be registered for event tracking, it would be great to add support for swiping, dragging and other custom actions. There may also be some work to be done around changing the `at-ng-event-tracking` directive to be element based instead of attribute based. It has an isolate scope that clashes with other directives on the same element if they have one too.
