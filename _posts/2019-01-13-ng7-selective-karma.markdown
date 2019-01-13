---
layout: post
slug: ng7-selective-karma
title:  "Running unit tests selectively with Angular7 (+ Ionic4) and Karma"
date:   2019-01-13 01:05:00
categories:
- tech
tags:
- angular
- angular 7
- karma
- webpack
---

Today with an Angular 7 / Ionic 4 project I was updating some unit tests, and this particular project I had been reasonably  self-disciplined about actually building out  unit test coverage. However the downside is that more tests take longer to compile, and run. This starts to get tedious when you only want to re-run a small number during development.

Out of the box, Angular 7 disappointed me significantly for the first time, by not providing a way to run individual unit tests from the cli. The solution involves leveraging a third party plugin, customising `angular.json` and Webpack and `test.ts`.

~ [Jump ahead to the solution](#solution) or the [recap](#recap)~

[![Selective Karma unit tests passing](http://blog.oldcomputerjunk.net/images/karma-screenshot-1.png)](http://blog.oldcomputerjunk.net/images/karma-screenshot-1.png)

---

_Yes, I'm using [Ionic4](https://beta.ionicframework.com/docs/) for essentially production code. In spite of still being in [beta](https://blog.ionicframework.com/announcing-ionic-4-beta/) it has just been so much a smoother epxerience with the build tools, possibly because they haven't completely obfuscated the whole process and instead call the ng tools directly. Admittedly, the first cab of the rank is actually a SPA webapp, but I have been impressed so far._

---

## Backstory

Now, having spent some years working with Java I'm very used to selecting individual test cases (down to single functions) with JUnit from the command line or from IDEs like Eclipse. And yet one thing that stuck me when I got back into Javascript a few years ago is how good the unit test infrastructure was, with tools like protractor (end to end), Karma, Jasmine, QUnit, istanbul (coverage mapping) and so-on.

So, I was a bit surprised when I discovered that out of the box you can't easily run a single or subset of tests using the Angular7 infrastructure.

---

*(For the uninitated: Angular 7 is the latest major release of Angular which is a somewhat opinionated web development framework that is best experienced using typescript, a typed superset of javascript. I'm mostly using it with Ionic 4 which is a "hybrid" framework that allows mobile applications to be developed using javascript (Angular 7 in this case) and then compiled into Android or iOS which run inside the native webview, using plugins to access native functions such as Bluetooth.)*

---

Now, it is ([or was](https://stackoverflow.com/a/31598188/2772465)?) [easy enough](https://medium.com/@bebraw/running-individual-tests-with-karma-mocha-89aece8ba18b) to run a [single test with Karma](https://stackoverflow.com/a/29151264/2772465), but this functionality [appears not to have been exposed through by Angular](https://stackoverflow.com/questions/40683673/how-to-execute-only-one-test-spec-with-angular-cli)  and I couldn't find anything in the Angular doco. The trick with a plain Karma project has been to use the environment [or command line](https://stackoverflow.com/a/29151264)  to specify a filter, which is then used to modify the webpack configuration for Karma. Angular 6+ significantly changed how webpack is used, this seems to have [broken](https://github.com/angular/angular-cli/issues/9949) a lot of [peoples work flows](https://github.com/angular/angular-cli/issues/10670).  In end I think actually the mechanisms are OK, just not obviously documented how to customise, and it does rely on a third party Angular builder; but in the process this makes it impossible to use the usual Karma tricks as well. [^1] [^2] [^3] [^6] [^9]

After some digging, it turns out there are github [issues](https://github.com/angular/angular-cli/issues/9026) that don't really help; I don't understand why such a fundamental feature such as running single unit test has been ignored. After all, after using it for a while the whole Angular 7 build system seems so much better than what I was dealing with for Ionic3.

The [commonly proposed](https://stackoverflow.com/a/40683791) but logistically inferior sugestion is to use the 'f' or 'x' prefixes of Jasmine.  For example, if Karma finds a test implemented using the [fdescribe](https://jasmine.github.io/api/2.8/global.html#fdescribe) function instead of `describe()`, where f stands for 'focus', then the other non-f tests are ignored. 'x' or `xdescribe()` is the reverse, meaning skip the test. **_This is fine as a temporary fix, but you don't wan't to commit it to git, or forget it was there or you end up with forgotten tests._** [^4]

Ultimately a proposed [solution](https://github.com/angular/angular-cli/issues/9026#issuecomment-354475734) that helped me find a way forward was to edit the `test.ts` file. [This is one of those obvious-in-hindsight things that makes it hard to google for](https://stackoverflow.com/a/50636750) By itself this is just as non-useful as the advice to use `fdescribe` ; however, along with some webpack magic I was able to come up with a sane solution. [^5] [^8]

## <a name="solution"></a> Solution

Lets start with the relevant code in `test.ts`, which is a file generated by the Angular build tools with a new project.
```
declare const require: any;

// First, initialize the Angular testing environment.
getTestBed().initTestEnvironment(
  BrowserDynamicTestingModule,
  platformBrowserDynamicTesting()
);

// Then we find all the tests.
const context = require.context('./', true, /\.spec\.ts$/);
// And load the modules.
context.keys().map(context);
```

Lets say we just want to run tests in `src/app/pages/something.spec.ts` The quick and dirty hack is to replace the regex:
```
const context = require.context('./', true, /something\.spec\.ts$/);
```

This gets us out of trouble, but it doesn't scale, and we badly don't want to commit that change!

So, how can we customise that regex dynamically?

The approach is to use a simple Webpack plugin, and modify the webpack, as per the next example, to substitute some text in the generated code.
This substitution then changes `test.ts` to:

```
declare const KARMA_SPEC_FILTER: any;
const context = require.context('./', true, KARMA_SPEC_FILTER);
```

**_(Note that when using Webpack to substitute text, it goes in verbatim)_**

and we might edit the karma configuration file `karma.conf.js` to add the Webpack text substitution plugin:
```
module.exports = function (config) {
  webpackConfig.plugins = (webpackConfig.plugins || []).concat(
    new webpack.DefinePlugin({KARMA_SPEC_FILTER: 'something.*\.spec\.ts$'}));

  ...rest of config is here...
```

**_However, there is no way out of the box to hook this into Angular 7._**


Eventually I [stumbled over](https://stackoverflow.com/questions/51068908/angular-cli-6-custom-webpack-config) `@angular-builders/custom-webpack`.
This wraps the default angular builder and lets you merge a webpack configuration. Install using `npm install --save-dev @angular-builders/custom-webpack`, and then edit `angular.json` as follows. [^7]

```
  "projects": {
    "app": {
      "root": "",
      "sourceRoot": "src",
      "projectType": "application",
      "prefix": "app",
      "schematics": {},
      "architect": {

...

        "test": {
          "builder": "@angular-builders/custom-webpack:browser",
          "options": {
            "customWebpackConfig": {
              "path": "./src/app-extra-webpack.config.js"
            },
            "preserveSymlinks": true,
```

Here, builder (for Ionic 4) previously was to `@angular-devkit/build-angular:browser` and has been changed to the custom webpack builder. Then specify the configuration file to merge into webpack, i.e. `customWebpackConfig`

Finally, the new file `src/app-extra-webpack.config.js` has the relevant part of webpack:

```
const FILTER = process.env.KARMA_FILTER;
let KARMA_SPEC_FILTER = '/.spec.ts$/';
if (FILTER) {
  KARMA_SPEC_FILTER = `/${FILTER}.spec.ts$/`;
}
module.exports = {
  plugins: [new webpack.DefinePlugin({KARMA_SPEC_FILTER})]
}
```

Putting it altogether now means we can run:
```
KARMA_FILTER='somefile-.*\.spec\.ts$' npm run test
```

and voila, we get our selective test execution.

## <a name="recap"></a> Recap

To allow control of Angular 7 Karma unit test selection from the command line - in this case using an environment variable - perform the following steps:

1. Install `@angular-devkit/build-angular:browser`
2. Create a custom webpack configuration merge that parses the environment or command line and creates a Webpack plugin with a desired regex
3. Modify `test.ts` to use the regex injected by Webpack
4. Modify `angular.json` to use the custom webpack configuration
5. Profit !!!
```
KARMA_FILTER='somefile-.*\.spec\.ts$' npm run test
```

---

[^1]: [https://stackoverflow.com/questions/40683673/how-to-execute-only-one-test-spec-with-angular-cli](https://stackoverflow.com/questions/40683673/how-to-execute-only-one-test-spec-with-angular-cli)

[^2]: [https://github.com/angular/angular-cli/issues/9949](https://github.com/angular/angular-cli/issues/9949)

[^3]: [https://github.com/angular/angular-cli/issues/10670](https://github.com/angular/angular-cli/issues/10670)

[^4]: [https://stackoverflow.com/a/40683791](https://stackoverflow.com/a/40683791)

[^5]: [https://stackoverflow.com/a/50636750](https://stackoverflow.com/a/50636750)

[^6]: [https://stackoverflow.com/a/29151264](https://stackoverflow.com/a/29151264)

[^7]: [https://stackoverflow.com/questions/51068908/angular-cli-6-custom-webpack-config](https://stackoverflow.com/questions/51068908/angular-cli-6-custom-webpack-config)

[^8]: [https://github.com/angular/angular-cli/issues/9026#issuecomment-354475734](https://github.com/angular/angular-cli/issues/9026#issuecomment-354475734)

[^9]: [https://stackoverflow.com/questions/50333461/karmawebpackmocha-running-single-tests](https://stackoverflow.com/questions/50333461/karmawebpackmocha-running-single-tests)


