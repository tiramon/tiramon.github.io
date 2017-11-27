# Creating a modal 'Loading' Dialog in Angular 2+
## Precondition
- You got a working Angular 2+ website

## The Problem

Not long ago I created a website that, after a click to navigate to another subpage, could take some time (around 3s) until all data was loaded. I used a [resolver](https://angular.io/api/router/Resolve) to load this data before the final page was shown, because it looked stupid when an empty page was loaded that filled with information after a while. This way the old page was shown until all data was loaded and the new page containing it was shown.
But the problem was until the data was loaded there was no feedback that the navigation and loading started. So i wanted to add a modal dialog that showed something like 'Loading...'

## The Solution
So i added a div to my main component template and styled it with [bulma](http://www.bulma.io/).
This looked like this
```html
<div class="modal" [class.is-active]="loading">
  <div class="modal-background"></div>
  <div class="modal-content has-text-centered">
    <div class="box">
      <h1 class="title"><span class="button is-loading" style="border: 0px;"></span>&nbsp;Loading...</h1>
    </div>
  </div>
</div>
<router-outlet></router-outlet>
```

And i changed my main component like this
```ts
import { Component } from '@angular/core';
import { Router, Event, NavigationStart, NavigationEnd, NavigationError, NavigationCancel } from '@angular/router';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss']
})
export class AppComponent {
  loading = true;

  constructor(private router: Router) {
    router.events.subscribe((routerEvent: Event) => {
      this.checkRouterEvent(routerEvent);
    });
  }

  checkRouterEvent(routerEvent: Event): void {
    if (routerEvent instanceof NavigationStart) {
      this.loading = true;
    }

    if (routerEvent instanceof NavigationEnd ||
      routerEvent instanceof NavigationCancel ||
      routerEvent instanceof NavigationError) {
      this.loading = false;
    }
  }
}
```

Now a little explenation for the code above. Whenever you navigate inside your Angular application (`router.navigate()`, `router.navigateByUrl()` or a `<a routerLink="/">Test</a>`) you use the RoutingModule of Angular. This creates `RouterEvent`s (`NavigationStart`, `NavigationEnd`, `NavigationCancel`, `NavigationError`, etc.). So when i receive the `NavigationStart` event i set my `loading` variable to true, because the Navigation just started. And when i receive a `NavigationEnd`, `NavigationCancel` or `NavigationError` i set `loading` to false, because if i don't set it to false on an error or cancel my application would stay locked behind the modal dialog and the page would have to be refershed to be accessable again.

So in the end this does nothing else to set a boolean based on routing events.

### The end
I hope i can help some people with this instruction. If anything is wrong, outdated, you simply know a better way or there is something else i forgot or people should be informed of just send me a pull request or create an issue https://github.com/tiramon/tiramon.github.io/issues
