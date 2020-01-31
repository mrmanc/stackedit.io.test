---
title: Everything's a component—writing domain specific, re-usable Angular components
  across squads
date: 2017-10-29 00:00:00 +01:00
tags:
- AngularJS
- Front End
layout: post
author: Craig Shipton
typescript-syntax: true
redirect_from:
- "/2017/10/29/eveythings-a-component-writing-domain-specific-reusable-angular-components-across-squads.html"
---

If you haven't heard about how Auto Trader works yet, we're structured into squads wherein each squad owns, maintains and develops within a particular domain to implement our business initiatives autonomously.  Within the retailer products division of Auto Trader are several squads working on the multi-faceted ‘Dealer Portal’ product to help vehicle dealers optimise their daily workings.  All of the disparate bits of technology to make Dealer Portal tick are encompassed under an umbrella project and common client/server technology stack we lovingly refer to as ‘Portal’. This post will discuss how we formed a strategy to maintain consistency across the Portal front-end.

At first inception Portal was a single API, with a monolithic single client:

![Legacy Portal]({{ site.github.url }}/images/{{page.date | date: "%F"}}/portal-api-legacy.svg){: .center-image }

Our current Portal structure has changed to be more split out and encompassing more Angular and comparatively less AngularJS:

![Portal]({{ site.github.url }}/images/{{page.date | date: "%F"}}/portal-api.svg){: .center-image }


Given that the different squads within the Portal umbrella project tend to have different requirements, it is common for large parts of the page to be custom and only relevant for that squad's customer base.  Even so, it is essential that we create a consistent look and feel across our apps to create a seamless experience for the end-user.  Perhaps the most obvious of commonality is the single header we have across Portal:

![Portal Header]({{ site.github.url }}/images/{{page.date | date: "%F"}}/portal-header.png){: .center-image }


At Auto Trader, we try to make the development of our products as simple as possible by choosing the right technology for our platform, audience, and developers.  For us working on retailer products, it's important that we can understand other squads' codebases to enable code review, idea sharing and being able to have fluency, familiarity, and efficiency when doing so.  With history proving that writing Portal in AngularJS worked well for us and the considerable work made by some very talented developers to create [ngUpgrade](https://angular.io/guide/upgrade), Angular would be our client-side framework of choice.  But with multiple squads working within their own streams under a common banner, how do we maintain a consistent look and feel while minimising redundancy?

We made our first Angular Component Library.


## What is the Component Library

The Component Library is a collection of Angular components, directives, pipes, and services made to be used within the apps created by squads at Auto Trader.  It is supported by the *Component Pattern Library*—a sibling project containing all of the SASS styles and patterns to make our components look the part.  As these libraries are specific to Auto Trader, all source is kept within our Github Enterprise repository for anyone in the company to read and contribute to.

### Ecosystem

Like most JavaScript client-side libraries written post-2010, the Component Library is built upon a Node.js foundation, built following the same principles as any other and hosted within our internal [NPM](https://www.npmjs.com/) repository.  Even though the library itself would not leave the confines of the Auto Trader ecosystem, it seemed pertinent for us to follow the standards set by the wider JavaScript and Angular communities to make integrating as frictionless as possible.  We've just not moved over to [Yarn](https://yarnpkg.com/) yet...

In terms of how it fits alongside existing component library solutions like Angular Material and ng-bootstrap—it actually sits on top of them!  We deliberately chose to build the Component Library this way as there is a world of developers who have already solved the problems that we would have if we created our own common components.  The open-source community can move much faster than us when it comes to maintaining these components and to put it bluntly, we'd rather contribute back to the open-source community with pull requests to existing libraries with new features we require.  Furthermore, we're not trying to solve generic components (each squad can use those directly) but instead more invested in solving problems unique to our domain.  In the same way that it's advantageous for apps to use existing domain-specific components from the Component Library, we can use the common components from those other libraries and style/adapt them to our needs.

Holistically, the library sits within this hierarchy:
	
![Auto Trader Application Hierarchy]({{ site.github.url }}/images/{{page.date | date: "%F"}}/angular-layers.svg){: .center-image }

As you can see, our apps are built upon a combination of the Component Library, generic component libraries, built-in Angular components and Angular itself.
	
## Component design

Being new to Angular itself when writing the library we read blog posts from other developers, experimented and iterated on an approach to writing components in a consistent and understandable manner.  We'll go over a few of the more interesting component development considerations that we made when writing our components.

### Sensible component state management

In the original AngularJS-based Portal, we had several cases of ‘`$scope` soup’ where a controller's `$scope` is passed through layers of directives creating fragile state, indeterminable behaviour and tight-coupling.  The fragility of the system created fear of change and thus a development nightmare.  This improved with the [‘controllerAs syntax’](https://johnpapa.net/angularjss-controller-as-and-the-vm-variable/) and developer experience with the framework.

Thankfully, Angular's clearly defined component structure encourages single responsibility and enables them to own their own data models making refactoring, testing, changing and reasoning about them so much easier.  We could take what we eventually learned with AngularJS's controllerAs syntax and apply it directly to Angular.


### ChangeDetectionStrategy.OnPush

In the earliest stages, we left change detection alone and we were happy to let the framework eagerly decide when components in our component tree should re-render themselves—mostly because it seemed like magic and we were not clear how it actually worked.  Our usage of the change detector completely changed after watching [this thorough presentation by Pascal Precht at NG-NL](https://www.youtube.com/watch?v=CUxD91DWkGM).

The most notable bits of the presentation in a nutshell:
- An ‘interaction’ is a [Zone.js patched event](https://github.com/angular/zone.js/blob/master/STANDARD-APIS.md) e.g. a user click, XHR request, a setTimeout etc.
- Any interaction with a component marks itself and all of its ancestors for changes—they all get checked for re-rendering regardless of the change detection strategy.
- Change detection begins from the root of the component tree and works its way down.
- For the default strategy, a component is always checked even if its inputs have not changed.
- For the OnPush strategy, only those components which have new input instances are checked—meaning that all component inputs must be immutable.


With the above in mind, `ChangeDetectionStrategy.OnPush` became the default for our components, albeit with some exceptions, for several reasons:
- `OnPush` gave the developer better control as to when change detection would happen and you can see exactly where developers expect change detection should be triggered by calls to `changeDetectorRef.markForCheck()` within their components.
- In situations where views didn't re-render when intuitively expected to, it gave developers opportunity to understand what was happening without settling on "it's just magic".
- In the same way that using immutable objects, `final` or `const` variables reduces the chance of mistakes in other programming languages, there's nothing to be lost over the default change detection strategy.
- This strategy is very similar to using immutable objects or final/constant variables in traditional programming languages in that it reduces the amount of acrobatics the developer can perform.  This has *big* reasoning benefits and simply encourages good practice.

Of course the most important of all is the improved rendering performance as whole component sub-trees can skip being checked when the user is interacting with a different part of the page!


### View Models

Given the desire to use the OnPush change detection strategy, it became evident that we would need to control how and when a component's inputs are changed.  Questions like "when does this component need to re-render itself?" and "what is the component responsible for showing when it does re-render?" prompted us to consider how we should put data into our components effectively.

We tried to be stringent in most cases on a component's only input being a ‘view model’ (a plain data object) which would represent everything that the component would display.  This would trickle down through a component's descendant tree to the bottom, with each component unwrapping the model within its template.  To enable this we would allow a component to interact with its direct children, but no further as to decouple our component hierarchy as much as possible.  Naturally, as we were using `OnPush`, any re-rendering of a component just required it to be given a new view model instance containing the change to be rendered.


For example, our `at-header` component would take a `Header` object containing two lists of navigation:

```ts
import { Navigation } from "./navigation";

export class Header {
  constructor(public primaryNavigation: Navigation, public secondaryNavigation: Navigation) {}
}
```

```ts
import { Component, Input } from "@angular/core";

@Component({
  selector: 'at-header',
  templateUrl: "./at-header.template.html"
})
export class AtHeader {
  @Input() header: Header;
}
```

Notice how the header component uses its own view model within `at-header.template.html` by unwrapping it and passing the model's two inner `Navigation` view models into the two navigation components:

```html
<at-navigation [navigation]="header.primaryNavigation"></at-navigation>
<at-navigation [navigation]="header.secondNavigation"></at-navigation>
```

Once the `at-navigation` components receive their `Navigation` view models, the header component has done its job and is oblivious to what the `at-navigation` components actually do with their view models—as should be the case when keeping components loosely-coupled and focused on one task.

### Component communication

Compared to view models (the *input*), we were less stringent on how components within the library communicated with each other (the *output*) as long as a general pattern was followed.  For most cases, child components telling their parents of events could be handled by simple void EventEmitters or ones carrying primitive payloads.  Using `at-navigation` from above as an example:

```ts
import { Component, Input, Output } from "@angular/core";

@Component({
  selector: 'at-navigation',
  templateUrl: "./at-navigation.template.html"
})
export class AtNavigation {
  @Input() navigation: Navigation;
  @Output() navigationClosed = new EventEmitter<void>();
  @Output() navigationClicked = new EventEmitter<string>();
  @Output() navigationChanged = new EventEmitter<Navigation>();

  clickedClose(): void {
    this.navigationClosed.emit();
  }

  clickedNavigationLink(url: string): void {
    this.navigationClicked.emit(url);
  }

  clickedHideNavigationLink(id: number): void {
    const changedNavigation = this.navigation.removeLinkWithId(id);
    this.navigationClicked.emit(changedNavigation);
  }
```

The parent could bind to it and perform whichever action is relevant for that situation.

In other cases, we found that passing up full objects representing a component's updated view model was a more suitable approach, like `clickedHideNavigationLink` in the example.  This worked well if a particular component (`at-navigation` in this example) didn't want its parent to decide how its own state should be changed when the user interacts with it.

	
### Component modules

With the majority of our components being very opinionated and requiring them to operate the same way across a number of apps, we made sure to make the most of Angular's module system.  Our feature modules would encompass everything required to use a particular component, e.g. the header, which would be made up of the modules owning that component's direct children.  Typically these child components would reside in sub-directories with their own feature module and so on, until reaching the bottom-most module—in the same way that components naturally formed into a tree structure, our modules would do the same.

In the following example, `at-header.module.ts` would declare `at-header.component.ts`, and directly import its dependent modules from the `at-header-navigation.module.ts`, `at-header-account.module.ts` and `at-header-item.module.ts` files.  `at-header-item.module.ts` would import the modules from `link`, `notification` and `submenu` etc.  This would mean a client needing to the use the `at-header` component would just import the module from `at-header.module.ts` and all required dependencies would already be available.

*Example component module hierarchy*
```
┠ at-header
  ┠ at-header.module.ts
  ┠ at-header.component.ts
  ┠ at-header-navigation
    ┠ at-header-navigation.module.ts
	  ┠ at-header-navigation.component.ts
  ┠ at-header-account
    ┠ at-header-account.component.ts
    ┠ at-header-account.module.ts
  ┠ at-header-item
    ┠ at-header-item.module.ts
    ┠ at-header-item.component.ts
    ┠ link
      ┠ ...
    ┠ notification
      ┠ ...
    ┠ submenu
      ┠ ...
┠ ...
```

The separation of components/modules from their children into sub-directories made navigating the library easy—understanding component ownership became even easier from looking at the directory structure.  Furthermore, it encouraged other developers to not be afraid of creating further sub-directories if necessary, easing the temptation to create monolithic components.


### Data service abstraction

A particular problem for us with opinionated modules is the retrieval of data to satisfy the view models that our components need to be rendered.  It would be inconvenient for each Portal client to each define their own version of the same service fetching the same data from the same API.  Even that being the case, adding a concrete data-retrieval service to our modules also felt like the wrong thing to do.  What happens when a client wants to fetch data from a different place, or if we want to change the mechanism by which we retrieve the data?

Our solution would be to define a TypeScript interface for the data service with an [InjectionToken](https://angular.io/api/core/InjectionToken) defined within the same file:

```ts
import { InjectionToken } from "@angular/core";
import { Observable } from "rxjs/Observable";

import { Vehicle } from "./domain/";

export interface VehicleLookupService {
  getVehicle(registration: string): Observable<Vehicle>;
}

export const VEHICLE_LOOKUP_SERVICE = new InjectionToken<VehicleLookupService>("vehicle.lookup.service");
```

With the interface defined, we can then create our concrete implementation.  In this example, our service hits an API over HTTP:

```ts
import { Injectable } from "@angular/core";
import { Response, URLSearchParams } from "@angular/http";
import { Observable } from "rxjs/Observable";
import { AuthHttp } from "angular2-jwt";

import { VehicleLookupService, Vehicle, LookupError } from "../../../at-vehicle-lookup/";

@Injectable()
export class PortalVehicleLookupService implements VehicleLookupService {

  constructor(private authHttp: AuthHttp) { }

  getVehicle(registration: string): Observable<Vehicle> {
    const params = new URLSearchParams();
    params.set("registration", registration);
    return this.authHttp.get("/api/vehicle-search", { search: params })
      .map(response => response.json().vehicles[0])
      .catch(error => {
        if (error instanceof Response) {
          if (error.status === 404) {
            return Observable.throw(new LookupError("vehicleNotFound", error));
          }
        }
        return Observable.throw(new LookupError("vehicleLookupFailure", error));
      });
  }
}
```

With the concrete implementation defined, we can assign it to the `InjectionToken` within our module:

```ts
import { NgModule } from "@angular/core";
import { CommonModule } from "@angular/common";
import { AuthHttp } from "angular2-jwt";

import { AtVehicleRegistrationLookup } from "./at-vehicle-registration-lookup.component";
import { PortalVehicleLookupService } from "./portal.vehicle.lookup.service";
import { VEHICLE_LOOKUP_SERVICE } from "../../../at-vehicle-lookup/";


@NgModule({
  imports: [CommonModule],
  declarations: [AtVehicleRegistrationLookup],
  exports: [AtVehicleRegistrationLookup],
  providers: [
    AuthHttp,
    { provide: VEHICLE_LOOKUP_SERVICE, useClass: PortalVehicleLookupService }]
})
export class AtVehicleRegistrationLookupModule {}
```

Components can use the `InjectionToken` construct within their own constructors, to get access to the concrete implementation:

```ts
import { Component, ChangeDetectionStrategy, Inject, ChangeDetectorRef } from "@angular/core";

import { VehicleLookupService, VEHICLE_LOOKUP_SERVICE } from "../../vehicle.lookup.service";
import { Vehicle } from "../../domain";

@Component({
  selector: "at-vehicle-registration-lookup",
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    ...
  `
})
export class AtVehicleRegistrationLookup {

  currentSearchedVehicle: Vehicle;

  constructor(@Inject(VEHICLE_LOOKUP_SERVICE) private vehicleLookupService: VehicleLookupService, private changeDetectorRef: ChangeDetectorRef) { }

  onSubmit(): void {
    this.vehicleLookupService.getVehicle(this.lookup.value.registration, this.lookup.value.mileage).subscribe(vehicle => {
      this.currentSearchedVehicle = vehicle;
      this.changeDetectorRef.markForCheck();
    }, error => {
      console.log(error);
    });
  }
}
```

With this approach, our components can remain completely oblivious as to where their data actually comes from while the modules they belong to change behind the scenes.  For a component library, where client apps can define their own implementations, this is very important as components can be used anywhere and in any context.

The only real downside with this approach is that with a vast number of interfaces there's potential for considerable boilerplate for client apps within a particular domain requiring repeated definition of the same concrete implementations.  We alleviated the problem by defining *service modules* (modules defining only concrete services) to provide collections of domain-specific, ready-made services for apps.

## Building the Component Library

As novices to Angular library writing originally, building the library took considerable time and lots of inspiration from other Angular node modules.  Our build process evolved many times over the past year while working on it and finally settled on:

1. Lint the new code using [Codelyzer](https://github.com/mgechev/codelyzer) with [TSLint](https://github.com/palantir/tslint).
2. Run all Karma unit tests in [webpack](https://webpack.js.org/) using [karma-webpack](https://github.com/webpack-contrib/karma-webpack), [karma-phantomjs-launcher](https://github.com/karma-runner/karma-phantomjs-launcher) (soon to be Chrome Headless via [karma-chrome-launcher](https://github.com/karma-runner/karma-chrome-launcher)), using the great test configuration from [AngularClass](https://github.com/AngularClass/angular-starter) as inspiration.
3. Apply a new version using [`npm version`](https://docs.npmjs.com/cli/version) according to [semver](https://docs.npmjs.com/misc/semver).
4. Remove any old code for distribution with [rimraf](https://github.com/isaacs/rimraf).
5. Copy all `.ts` and `.html` files into temp directory with [copyfiles](https://github.com/calvinmetcalf/copyfiles).
6. Transpile `.ts` files within temp directory using either
	- `tsc`: the TypeScript compiler if transpiling to ES5 JavaScript to produce `.js` and `.d.ts` files.
	- `ngc`: the Angular compiler if transpiling to [Ahead-of-Time](https://angular.io/guide/aot-compiler) compatible ES6 JavaScript to produce `.js`, `.d.ts` and `.metadata.json` files.
7. Inline SVGs into the HTML templates and inline HTML templates into components within temp directory using the [inline resources script](https://github.com/angular/material2/blob/master/tools/package-tools/inline-resources.ts) from Angular Material 2.
8. Move all generated `.js`, `.d.ts` (and `.metadata.json` if applicable) files into dist.

With these steps complete we can package our dist folder ready for publishing to our internal NPM repository for other squads to use as they see fit.


## Evaluation of approach

At the time of creation, the approach of writing an opinionated library to solve the problems that we faced was either non-existent or not written about.  Over the past year, we identified a number of advantages and disadvantages with our solution. 

### Advantages

After getting the library ready to be used in production, we found that our approach had the following advantages:

- A library is good common ground to set coding standards.
- As Angular is modular, you can write a component once, add it to a module and push out to everybody else with minimal effort.
- Updates to shared components can benefit everyone.
- It is a great proving ground for testing new Angular features and processes for new developers, without getting caught up with build system intricacies.
- We chose to keep styles abstracted away, meaning that we can easily create further libraries for other frameworks, whether that's React, Web Components, Vue.js etc., using the same styles.
- Development and distribution of the library within Auto Trader proved that shared libraries were possible.
- With a number of shared components and sufficient pre-made data services, we were able to create a starter project that gets new apps created very quickly.

### Disadvantages

Even with the above benefits we've also found several drawbacks that we've yet to solve:

- Angular requires modules to be told exactly what other modules, services and components they will be using, whether loaded eagerly or not, without considerable effort.  This does mean there has to be repeated boilerplate across app modules.
- Finished components still need to be maintained—a problem that would be solved by the large community in an open-source context.
- Component ownership can be difficult to manage as there's often a business assumption that component creators are the maintainers.
- The time it takes to get even something basic running takes time, especially so if you're new to Angular and its dependencies.
- Propagating new library versions across several apps simultaneously remains a challenge.  In an environment where all apps need to build fresh webpack bundles containing new library code, we're limited on options.  We're looking into making use of semantic versioning to know when we can automatically propagate library changes to client apps.   This is probably our biggest hurdle in making this library.


## Recommendations

Taking all of our experience into consideration we'd make some key recommendations should you decide to take the approach that we did.  Firstly, if building a library to be used across apps and across build systems, it's best to distribute raw transpiled source like any other JavaScript library would.  In our naivety, we first tried distributing a webpack bundle per component but we ended up shipping too much to every client—even if the webpack boilerplate added wasn't an issue, some clients may not want absolutely everything in that bundle.

When creating a component that you *know* will be shared, define your component's view model and outputs first.  By defining your component's API first, you can develop the component without worrying about the context in which it may be used and just concentrate on satisfying your defined API.  Conversely, when creating a component you *suspect* will be shared, develop it in your project and only move it over for the second usage of it—it may never happen or your version may be too domain-specific to be re-used.  It may even need re-engineering in order to generalise.

A sandbox is a good environment for tinkering with work-in-progress components which aren't quite deployable and is worth the initial time investment to improve the speed of the test-develop cycle.

Don't be afraid of abstracting what data you need from how you get it, by defining data service interfaces with Angular's InjectionToken mechanism.  If you end up with lots of interfaces you can create service modules to define concrete service implementations relevant to a particular domain.

Most important of all is to ensure that you have a mechanism in place that can automate the propagation of new component changes across apps as this has been one of the biggest roadblocks in our implementation of the component library.
