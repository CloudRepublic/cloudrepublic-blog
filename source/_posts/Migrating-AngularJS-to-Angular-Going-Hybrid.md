---
title: Migrating AngularJS to Angular - Going Hybrid
date: 2019-01-11 14:28:20
tags: AngularJS, Angular, Webpack, Hybrid
---
So, you are finally ready to start to migrate your old AngularJS to the new fancy Angular 7. But it is kind of big and you cannot afford not to release for half a year. So how are you going to do it? You want to upgrade to the latest Angular version, but you can’t do it with a big bang. Well there is a great hybrid solution made by the Angular team called @Angular/Upgrade. It basically lets you run AngularJS inside Angular and makes all you’re AngularJS Services, Components, Directives etc. available in Angular and vice versa. 	
## Getting your AngularJS Code ready
First, we need to get our source code and build process ready. Secondly, we need to switch to webpack, this will make the rest of the steps way easier. Because every application has different needs I won’t go into detail on how to do this. But for those of you who’ve never used Webpack before, [this quickstart](https://webpack.js.org/guides/getting-started/) will give you a head start.

If you haven’t done so already install all your dependencies with a package manager like NPM, this will save you a lot of headaches while merging AngularJS with Angular.

The last step is to migrate all JavaScript files to typescript. Although recommended it is only required if you are planning to migrate your existing AngularJS code to Angular.

`rename *.js *.ts`

Should do the trick on windows.
To use TypeScript, you need to add TSConfig file, a basic example is found below.
```json
{
  "compilerOptions": {
    "noImplicitAny": false,
    "noEmitOnError": true,
    "removeComments": false,
    "emitDecoratorMetadata": true,
    "sourceMap": true,
    "target": "es5",
    "module": "es2015",
    "moduleResolution": "node",
    "experimentalDecorators": true,
    "noStrictGenericChecks": true,
    "lib": ["es2018", "dom"],
    "suppressImplicitAnyIndexErrors": true,
    "rootDir": "src",
    "outDir": "dist",
    "baseUrl": "./"
  }
}
```
Optionally you could add type definitions for AngularJS and other dependencies this will add type hint’s for your old AngularJS code.
## Installing Angular
So the first step is to installing the Angular Framework. Just run the following NPM command and your good.

```bash
npm i –save @angular/common @angular/compiler @angular/core @angular/forms @angular/http @angular/platform-browser @angular/platform-browser-dynamic core-js zone.js
```
We need to add one more dependency. This package will handle the communication between AngularJS and Angular.

```bash
npm i –save @angular/upgrade
```
## Setting up the hybrid
Now that we have everything installed we can start coding. First add an file called app.module.ts to your source root. This will be our root angular module and will bootstrap AngularJS.

```ts
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { UpgradeModule } from "@angular/upgrade/static";

@NgModule({
  imports: [BrowserModule, UpgradeModule],
})
export class AppModule {
  constructor(private upgrade: UpgradeModule) {}
  public ngDoBootstrap() {
    // bootstrap(angularJs root element in dom, AngularJS root module, bootstrap options)
    this.upgrade.bootstrap(document.body, ["app"], { strictDi: true });
  }
}
```

Angular doesn’t magically know that this is the root module. The code to tell Angular this is usually placed in a file called main.ts, this file needs to be added as an entry in the webpack configuration.

```ts
import "./app"; // AngularJS code
import { platformBrowserDynamic } from "@angular/platform-browser-dynamic";
import { AppModule } from "./app.module";

platformBrowserDynamic().bootstrapModule(AppModule);
```
Finally, we need to add some Polyfills. Usually this is added in a separate file and added to the webpack config as an entry. So, let’s add a polyfills.ts
```ts
import "core-js/es6/reflect";
import "core-js/es7/reflect";
import "zone.js/dist/zone";
```
That’s it. You are running in hybrid.
## So, what’s next
There is not one way to upgrade your project. If you’re planning to migrate your existing AngularJS components to Angular you should take a look at upgrading AngularJS components to work with Angular [over here](https://angular.io/api/upgrade/static/UpgradeComponent) and Downgrading Angular Components to work with AngularJS [over here](https://angular.io/api/upgrade/static/downgradeComponent). These will allow you to keep upgrade a component without breaking the stuff around it.
But if you are on an older code base that uses separate controllers and html templates, you should rewrite your code to a component-based system. A tool that could help you is the hybrid library for UI-Router. This will allow you to upgrade page by page. You can find this library [over here](https://github.com/ui-router/angular-hybrid).

** This blog is written by: ** 
Martin Ligtenberg
<script src="//platform.linkedin.com/in.js" type="text/javascript"></script>
<script type="IN/MemberProfile" data-id="https://www.linkedin.com/in/mvligtenberg/" data-format="inline" data-related="false"></script>
