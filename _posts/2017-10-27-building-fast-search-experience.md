---
layout: post
title:  "Building a Fast Search Experience"
author: Elliot Sumner
---

Auto Trader provides a search platform for dealers to buy vehicles from other traders. A high-performance search experience is critical, as this helps create a competitive marketplace for dealers to purchase vehicles. This blog post will take you through some of the changes we made to create a high-performance search platform that regularly returns results in less than a second.

The default sort order is newly listed vehicles. This platform behaves very differently from the normal site you may be familiar with. The search experience is geared around our dealers being able to rapidly decide on vehicles to purchase to fill their forecourt. Dealers can then further refine their searches to show vehicles that may be suitable for them. A slower site means fewer searches and therefore less incentive for dealers to advertise with us. 

### Early Release
 
When we initially launched the new platform there were still some changes we had to make to make the search experience acceptable. Although our networking calls were returning in a fairly reasonable time (around 1.5 seconds) we noticed that it was taking a long time to start searches and display the results.

![old search]({{site.github.url}}/images/2017-10-27/old-search.png)

 
This was due to some known performance limitations on how AngularJS handles changes to page contents. There were a few key changes that we made that vastly improved the performance of the single page application:

A lot of the code was initially written using the ng-show directive. This directive conditionally hides or shows items on the page, however, this is just a simple visibility change to the contents of the DOM. The content is still there and still has to be processed for changes. 
 
```html
<div ng-show="false" style="visibility: hidden">
    <span>This Content is still present</span>
</div>
```
 
We changed the user input components to use ng-if instead. This would conditionally remove the content from the DOM, reducing the amount of work AngularJS would have to do when a change happened. 

```html
<div ng-if="false">
    <!-- <span>This Content has been removed</span> -->
</div>
```

This was a particularly acute problem on the multi-select for selecting makes of cars, as it meant that AngularJS would need to watch four variables for each element, resulting in nearly 1000 additional checks needed for each change to the page.

```html
<label>{ {itemLabel} }</label>
<div ng-if="displayed">
...content...
</div>
```

Within the application, there were many parts of the content that were not modified more than once. By changing these items to be one time bindings we could signal to AngularJS that these items need not be processed any further.

```html
<label>{ {::itemLabel} }</label>
<div ng-if="displayed">
...content...
</div>
```

By carefully applying these optimisations we were able to reduce the number of watched variables from several thousand to around 800 resulting in a more rapid AngularJS digest cycle.

We eventually ended up with a performance profile like the following:

![initial loading]({{site.github.url}}/images/2017-10-27/initial-loading.png)

Even after the loading phase, it can be seen that there is a lot of time spent with the browser processing the server results.

### UX Improvements

As one of the core parts of the sourcing user experience, we decided that it needed a revamp taking into account user feedback and to introduce our new brand. Our designer saw this as a great opportunity to really look at how users were using the platform and make it fit the user's needs.

![new search]({{site.github.url}}/images/2017-10-27/new-search.png)

We looked at anonymous data from Google Analytics, and heatmaps from Hotjar to see how users were interacting with the existing search. This allowed us to create and iterate on a design so that the most commonly used items were the most accessible. The heatmaps showed us that some heavily used user interface elements, such as the make and model of a vehicle, were not even on the screen when the majority of users came to the search.

![old heatmap]({{site.github.url}}/images/2017-10-27/old-heatmap.png)

The most commonly used items were moved to the top of the search screen, and the items on the side were reordered and reworked to be in the optimal order. We added support for multi-selects where possible to allow users to search for all they wanted in one go, rather than performing multiple searches. In addition, we looked at the Google Analytics data and spoke to users of the system to see what features were missing. This allowed us to discover features that we had missed, which are now some of our more heavily used filters.

![new heatmap]({{site.github.url}}/images/2017-10-27/new-heatmap.png)

A heatmap for the updated search shows a significant improvement in usability.

#### Pre Emptive Search

The search experience still forced users to wait for a result to come back before they could perform another search. This was due to the fact that the contents of the filters are driven from the server side. For example, we would not show the Petrol fuel type for users that searched for Tesla vehicles.

![new heatmap]({{site.github.url}}/images/2017-10-27/normal-search.png)

To make the user interaction more responsive, we could allow users to change the selection of the last item they selected while waiting for the results to come back from the server. We called this pre-emptive search.

![new heatmap]({{site.github.url}}/images/2017-10-27/preemptive-search.png)

This would help users who wanted to select multiple items of the same type, e.g Petrol, and Hybrid. We thought that this would be a big improvement to usability, however, when attempting to use it on mobile devices we noticed that the multi-select options were still not very responsive.

![mobile loading]({{site.github.url}}/images/2017-10-27/mobile-loading.png)

The analysis showed that the loading bar on mobile devices was causing the browser to reprocess all the watchers on the page. As this was all done on the UI thread it meant that the server would generally return a response before the browser would let the user select another option.

By changing the loading spinner so that it would only be displayed after a few seconds we freed the browser up to listen to new user interactions.

![mobile loading]({{site.github.url}}/images/2017-10-27/mobile-loading-no-progress.png)

### Updated Listings


As a late-stage addition to the updated search experience, the search listings were redesigned. During this time we also converted our project to a hybrid AngularJS/Angular application. As part of this, we decided that it would be worth rebuilding the listings as an Angular component (rather than AngularJS) in TypeScript, as it would simplify maintainability and allow us to remove a large amount of legacy AngularJS code.

During the development process, it was noticed that the new listings were faster than the legacy AngularJS listings. This was expected, as the listings have a large number of elements and complex logic. We thought it was worth investigating how much faster we could make the new listings.

We discovered that the only thing that needed checking on the card was a loading spinner state, displayed when we are waiting for a result to come back from the server. The listings cards are immediately replaced by new ones when new results come in.

Knowing this we decided to switch the change detection to manual. This would allow us to decide exactly when a card update would happen.


```ts
@Component({
    selector: 'search-listings',
    templateUrl: './search-listings.template.html',
    changeDetection: ChangeDetectionStrategy.OnPush // <-- Set to manual
})
class SearchListingsComponent {
    @Input()
    set loading(loading:boolean) {
        let oldLoading:boolean = this.loading;
        this.loading = loading;
        if(loading != oldLoading) {
            this.changeDetector.detectChanges(); // <-- explicitly notify that a change has happened
        }
    }
}
```

By doing this, and also changing how our global app loader triggers we were able to take the post-search processing down from around 800ms to 350ms. The overall results for the new listings were encouraging. Through A/B testing we saw a 5% overall improvement in engagement, and a 25% improvement on mobile devices. 

#### Leveraging the JIT Compiler

Moving to a hybrid application allowed us to enable AOT template compilation. This helped with the initial load times of the project. See [this blog post]({{site.github.url}}/2017/07/24/how-we-halved-page-load-times.html) about the improvements that AOT compilation can bring.

An additional unexpected improvement was when the browser's JIT compiler activates on the listings card. As the HTML template is compiled into JavaScript, the browser's runtime engine can perform extensive optimisation. This will generally happen after a few searches, something that can't happen in the AngularJS world. On Chrome, this can take the listings processing down to around 150ms. This post goes into details on 
[firing up ignition interpreter](https://v8project.blogspot.co.uk/2016/08/firing-up-ignition-interpreter.html).

![jit loading improvements]({{site.github.url}}/images/2017-10-27/jit-loading-improvements.png)

For more recent versions of Chrome the code can be cached, further improving the application startup time and allowing users to come back to the site performing at its maximum speed.

### Conclusion

This post demonstrates some of the changes we did to improve our search experience. Combined with a number of optimisations on the back end to improve the performance of trade search, we were able to get the search results to appear below one second, even when searching through nearly half a million vehicles. In a future post, we will look at what changes were needed on the server side to improve search performance.

*[AngularJS]: Sometimes known as Angular 1. The legacy version of Angular
*[Angular]: Sometimes known as Angular 2+. This is written in TypeScript
*[AOT]: Ahead Of Time
*[JIT]: Just In Time
 
