# Using Piwik in an Angular 2+ Website

## Preconditions
- You got Piwik installed, configured and running
- You got a working Angular 2+ Website

## The Problem
Piwik is design for normal websites. Every time the user navigates to another page the Piwik JavaScript is loaded via the script tag in the had and a request is send to the Piwik server. So far so good... but Angular is a SPA ([Single-page-application](https://en.wikipedia.org/wiki/Single-page_application)). So on the initial start of the site piwik is called as on any other webpage, but on any further site intern navigation the page is not reloaded and so the piwik script is also only called on the initial request.

## Solution
After a while of searching i found this website with a nice instruction on how to get piwik running in a SPA 
https://lmu-pms.github.io/irom-blog/posts/Angular2WithPiwik.html
But there are some flaws in the instruction at this website and on the referenced library site, so i will write my own short instruction.

### 1. Install Angular2Piwik
Angular2Piwik is a simple wrapper for the piwik api.

According to the instruction, their is also another library [Abgulartics2](https://github.com/angulartics/angulartics2) which also supports other analytics provider, but like the writer of the other instruction i don't want all that overhead for the other providers. So i also use [Angular2Piwik](https://github.com/awronka/Angular2Piwik).
```console
npm install --save angular2piwik
```

### 2. Include Angular2Piwik in your Website
First include the module to your main module.
```ts
// app.module.ts
// ...
import { Angular2PiwikModule } from 'angular2piwik';
// ...
@NgModule({
  imports: [
    // ...
    Angular2PiwikModule
  ],
  // ...
})
```
Here is the first difference to the other instructions. In the other instructions the always import like this
```ts
import { Angular2PiwikModule } from 'Angular2Piwik';
```
This works find on any non case sensitive os like windows. But it took me quite some time to figure out why it didn't work on my unix machine.
The reason is that the entry in the node_modules folder is `angular2piwik`. Windows doesn't care for case and has no problem finding the right folder, but linux search for `Angular2Piwik` where only a àngular2piwik` does exist ... can't work.

For the following step there are 2 ways to install and configure piwik to your site.
#### 2.1 Include the Piwik JS the usual way
In your Piwik installation you can get a tracking code snippet for a website. Install it as last entry in the head script of your `index.html` as recommmended by Piwik.
#### 2.2 Configure it in typescript
Not sure if this works ... havn't tried it yet and there is a issue on the Angular2Piwik website that claims the documentation is incomplete

You can configure your Piwik in your main component.
```ts
import { Component } from '@angular/core';
import { InitializePiwik } from 'Angular2Piwik';

@Component({
  selector: 'app',
  providers: [ initializePiwik ],
  template: `<router-outlet></router-outlet>` // Or what your root template is.
})
export class AppComponent {
  constructor(
    private initializePiwik: InitializePiwik
    ) {
      const url = `//*************:*****/anayltics/`; // set your url to whatever should be communicating with Piwik with the correct backslashes
      initializePiwik.init(url);
    }
}
```

### 3. Setup a central tracking
On the other instructions they say that the `trackPageView()` method should be called in any components that you want to track.
I got a few problems with this approach:
1. alot of work
Every time i add a new component i have to call this method. If i forget it the page does not get tracked. I have to configure `setDocumentTitle`, `setReferrerUrl` and `setCustomUrl` in every component.
2. components from libraries
As long as i only want to track components i wrote myself there should be no problem, but since i can't add the method call to components from libraries ...
3. multiple calls
Very soon i recognized that with one click my piwik reported that i just visited the same site multiple times. I had multiple components stacked and called the `trackPageView()` method multiple times

So i create my own centralized approach. So i only have to write a configuration once and don't have to care what site are called. They are all tracked automatically.

Here a little explenation to the following simple main component.
Whenever you navigate inside your Angular application (`router.navigate()`, `router.navigateByUrl()` or a `<a routerLink="/">Test</a>`) you use the RoutingModule of Angular. This creates `RouterEvent`s (`NavigationStart`, `NavigationEnd`, `NavigationCancel`, `NavigationError`, etc.). For us only the `NavigationEnd` event is important. At this step we can say that we just navigated inside our application. In the code below we always store the url we arrived at so we know the referrer url for the next navigation. We also refresh the configuration with the current document title and the current url.
After all this configuration is done we tell the library to send all known information to our Piwik server by calling the `trackPageView()` method.
```ts
import { Component } from '@angular/core';
import { NavigationEnd, Router } from '@angular/router';
import { ConfigurePiwikTracker, UsePiwikTracker } from 'angular2piwik';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss'],
  providers: [UsePiwikTracker]
})
export class AppComponent {
  previousUrl: string = null;
  constructor(
    private router: Router,
    private configurePiwikTracker: ConfigurePiwikTracker,
    private usePiwikTracker: UsePiwikTracker
  ) {
    router.events.filter(event => event instanceof NavigationEnd).subscribe( 
      routerEvent => {
        this.configurePiwikTracker.setDocumentTitle();
        if (routerEvent instanceof NavigationEnd) {
          if (this.previousUrl) {
            this.configurePiwikTracker.setReferrerUrl(this.previousUrl);
          }
          this.configurePiwikTracker.setCustomUrl(routerEvent.url);
          this.previousUrl = routerEvent.url;
          this.usePiwikTracker.trackPageView();
        }
    });
  }
}
```

At the page i used this first at users can only do stuff after they loged in to my page. Because i like to have some more information i added a few lines to my [AuthGuardService](https://angular.io/api/router/CanActivate).
I just added a AuthGuard that checks it the user is logged in or not (i used an OpenIdConnect library for this)

So if the current user has a valid token or valid IdentityClaim i set the name as Piwik userid. If the user is not loged in correctly i set the userid to null.
```ts
import { Injectable } from '@angular/core';
import { ActivatedRouteSnapshot, CanActivate, Router, RouterStateSnapshot } from '@angular/router';
import { OAuthService } from 'angular-oauth2-oidc';
import { ConfigurePiwikTracker } from 'angular2piwik';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private oauthService: OAuthService,
    private router: Router,
    private configurePiwikTracker: ConfigurePiwikTracker
  ) {
  }

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): boolean {
    if (this.oauthService.hasValidIdToken()) {
      this.configurePiwikTracker.setUserId(this.oauthService.getIdentityClaims().name);
      return true;
    }
    if (this.oauthService.getIdentityClaims()) {
      this.configurePiwikTracker.setUserId(this.oauthService.getIdentityClaims().name);
      return true;
    }
    this.configurePiwikTracker.setUserId(null);
    this.router.navigate(['/home'], {queryParams: {url: state.url }} );
    return false;
  }
}
```

### The end
I hope i can help some people with this instruction. If anything is wrong, outdated, you simply know a better way or there is something else i forgot or people should be informed of just send me a pull request or create an issue https://github.com/tiramon/tiramon.github.io/issues
