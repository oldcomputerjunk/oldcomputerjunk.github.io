---
layout: post
slug: ionic4-from-ionic3-deploying-as-a-web-app
title:  "Some notes on moving to Ionic4 from Ionic3 and building a web app"
date:   2019-02-01 01:00:00
categories:
- tech
tags:
- angular
- angular 7
- ionic4
- ionic
---

Ionic4 has recently been released out of beta, but I've been using the beta version for a while now.
As things have stablised here I have succesfully deployed an Ionic4 application as a web application.

The app itself is not yet ready for public access so I can't link to it here 😎

### Ionic versioning confusion ###

Ionic4 has just been released, and this overhauls and streamlines both the build system, and the components (UI) framework.
Although coupled in theory you can use the component framework without the build system, and a big new feature is you
don't have to use Angular though it is the default.

However the ionic cli (npm package `ionic`) was released as 4.0.0 some time ago and is now at 4.10.1, this seems to be very independent of IOnic4 itself.

This caused problems when it the cli itself tripped to 4.0.0 and ionic proper was still at ~3.18, and I discovered that it broke my existing Ionic3 deployments,
I guess to to immature changes in support of the Ionic4 beta. A few months after that I discovered again that they had been fixed so
now it is possible to use the ionic cli >= 4.4.0 to build a (now legacy) Ionic 3 native app without problems.

One thing I will typically do is install the ionic cli as a dev dependency into my project. It does appear that the global cli if present will use this (unconfirmed)
or you can simply use `npx ionic` to run it instead.

Confused yet? I was. Here is a table that attemps to help, it includes some other related things as well.
This is my own understanding, the ultimate source of truth would be the project [github page](https://github.com/ionic-team/ionic).

| Product            | License | npm packages and versions    | status             | Notes |
| ------------       | ------- | -------------------------    | ------------------ | ----- |
| Ionic cli          |  MIT    | `ionic            @~4.10.1`  |                    |
|                    |         |                              |                    |
| Ionic4             |         | `@ionic-native/…   ^@5.0.0`  | (just out of beta) |
| type=angular       |         | `@ionic/angular/…  ^@4.0.0`  | (just out of beta) |
|                    |         | `@angular/…        ~@7.2.2`  |                    |
|                    |         | `typescript        ~@3.1.6`  |                    |
|                    |         |                              |                    |
| Ionic3             |         | `@ionic-native/…  ^@4.19.0`  | (legacy)           | NPM outdated now shows 5.0 but I havent tested to see if it works
| type=ionic-angular |         | `@ionic/app-scripts @3.2.1`  | (legacy)           | `app-scripts` is not needed for Ionic4/Angular7
|                    |         | `ionic-angular      @3.9.2`  | (legacy)           | `ionic-angular` is not needed for Ionic4/Angular7
|                    |         | `@angular/…       ~@5.2.11`  | (legacy)           | Last of Angular5 series - upgrading will break Ionic3 application
|                    |         | `typescript        ~@2.8.4`  |                    | The default is actually 2.6 but I managed to update this far succesfully
|                    |         |                              |                    |

### Comparing Ionic4 and Ionic3 when used to build a web app (Single Page Application) not intended to be native ###

The first full project I built using Ionic4 was a web app. Given I started this a few months ago there was some risk in using
a beta framework for production, using it for a web app not quite due for release was a good trade off, as I'd have more control over its
deployment and the ability to immediately rectify issues for all users.

I mentioned this previously but I think Ionic4 is much improved in its architectured, particularly in how for Angular apps
it no longer replaces what the Angular7 cli does but simply calls it, meaning the code is essentially an Angular7 app with some
additional tooling and the Ionic component and native wrapper libraries as needed.

One side effect of this I learned is that AoT (angular Ahead of Time) is kind-of automatic when you build in `--prod` mode and further, the same code is produced
where you run `ionic build` or `ionic cordova build browser`. Previously, I don't remember whether the code built in `./www` was the same
as `./platforms/browser/www` but I seem to recall having to use the latter if I wanted a deployable wep app that actually worked.
Now though, having tested both if you run `diff -r www platforms/browser/www` the only difference was that the `config.xml` file additionally
gets copied to `platforms/browser/www`.
Whether this holds in all cases remains to be seen - yet is possible that this is due to the fact I introduced no native plugins into this code base
(the defaults statusbar and splashscreen were left in but the Javascript side of those already stubs itself out when not a native build)

### Browsers and Poplyfills and so on ###

I'm still testing the deployment over a menagery of browsers.

Previously I had to have a whole manner of ployfills over and above what came in a default Ionic3 project, although those were all first
created over a year ago so perhaps the defaults had improved since. Also, a larger percentage of browsers support ES5 out of the box.
(ES5 is the version of javascript and related APIs they can execute without need a polyfill; a polyfill is a piece of code that emulates missing functionality,
usually incurring performance degradation or sometimes not quite 100% compatible)

So far I have avoided having to do this. In part this might also be due to improvements to typescript and the settings.

In particular, I have typescript set to target ES5, which catches almost everything now. This is a default for a new ionic project as well.

The following defaults were generated using `ionic start testionic4 blank --type=angular --cordova --no-link` - although cordova is not strictly
needed for a non-native app I always include it becase the default plugins shim out and so far I've wanted to make it easier to convert to
a native app if required.  The use of `--no-link` avoids trying to connect to the Ionic SAAS product Appflow which I don't use presently.

Ionic4 sets up the following `tsconfig.json` compiler options, augmented for my needs:

```
  …
  "compilerOptions": {
    "baseUrl": "./",
    "outDir": "./dist/out-tsc",
    "sourceMap": true,
    "declaration": false,
    "module": "es2015",
    "moduleResolution": "node",
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "forceConsistentCasingInFileNames": true,   <-- added by me
    "allowSyntheticDefaultImports": true,       <-- added by me
    "target": "es5",
    "typeRoots": [
      "node_modules/@types"
    ],
    "lib": [
      "es2018",
      "dom"
    ]
  }
  …
```

Here, `allowSyntheticDefaultImports` will let you do the `import something from 'module';`
instead of `import * as something from 'module';`, for javascript modules without type information.
The setting `forceConsistentCasingInFileNames` will cause errors if you have two files with the same name with different case,
which can cause a world of pain if you need to run a build on a Windows platform. Presently I'm alternating between Linux and MacOS but
from time to time I do need to use Windows as well. In any case I think just from a human confusion perspective
I think having  both `helloWorld.ts` and `helloworld.ts` in the same place is not a good idea...


### deployment

Is easy!

You can test locally using the node web server if installed - browse to http://localhost:8080 after running:
```
ionic build --prod
http-server ./www
```

If everything is good you can zip up or otherwise copy `www` to a suitable web server.

Because I'm using google firebase as a database for this app I deploy to google cloud hosting.
This lets me add a one liner to `package.json`, `"deploy": "ionic build --prod && firebase deploy --only hosting"`
(Note there is some configuration required to set up `firebase deploy` so it knows to use `./www`, etc.)
