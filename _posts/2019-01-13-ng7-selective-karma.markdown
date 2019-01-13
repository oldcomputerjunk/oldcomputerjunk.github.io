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

Today with an Angular 7 / Ionic 4 project I was updating some unit tests, and this particular project I had been reasonably  self-disciplined about actually building out  unit test coverage. However the downside is that more tests take longers to compile, and run. This starts to get tedious when you only want to re-run a small number during development.

Yes, I'm using [Ionic4](https://beta.ionicframework.com/docs/) for essentially production code. In spite of still being in [beta](https://blog.ionicframework.com/announcing-ionic-4-beta/) it has just been so much a smoother epxerience with the build tools, possibly because they haven't completely obfuscated the whole process and instead call the ng tools directly. Admittedly, the first cab of the rank is actually a SPA webapp, but I have been impressed so far.

Out of the box, Angular 7 disappointed me for the first time since using it by not providing a way to run individual unit tests from the cli. The solution involves a third party plugin, customising `angular.json` and Webpack and `test.ts`.

## Backstory

Now, having spent some years working with Java I'm very used to selecting individual test cases (down to single functions) with JUnit from the command line or from IDEs like Eclipse. And yet one thing that stuck me when I got back into Javascript a few years ago is how good the unit test infrastructure was, with tools like protractor (end to end), Karma, Jasmine, QUnit, istanbul (coverage mapping) and so-on. So I was a bit surprised when I discovered that out of the box you can't easily run a single or subset of tests using the Angular7 infrastructure.

(For the uninitated, Angular 7 is the latest major release of Angular which is a somewhat opinionated web development framework that is best experienced using typescript, a typed superset of javascript. I'm mostly using it with Ionic 4 which is a "hybrid" framework that allows mobile applications to be developed using javascript (Angular 7 in this case) and then compiled into Android or iOS which run inside the native webview, using plugins to access native functions such as Bluetooth.)

Now, it is (or was?) easy enough to run a single test with Karma, but this functionality has not been exposed through by Angular [^1]. The trick with a plain Karma project was to use the environment or command line to specify a filter, which is then used to modify the webpack configuration for Karma. Angular 6+ significantly changed how webpack is used, this seems to have broken a lot of peoples work flows. [^2], [^3] In end I think actually the mechanisms are OK, just not obviously documented how to customise, and it does rely on a third party Angular builder; but in the process this makes it impossible to use the usual Karma tricks as well.

After some digging, it turns out there are some long-time open issues on Github (fixme); I don't understand why such a fundamental feature such as running single unit test has been ignored. After all, after using it for a while the whole Angular 7 build system seems so much better than what I was dealing with for Ionic3.

The commonly proposed but logistically inferior sugestion is to use the 'f' or 'x' prefixes of Jasmine. (https://stackoverflow.com/a/40683791, but repeated elsewhere) For example, if Karma finds a test implemented using the `fdescribe()` function instead of `describe()`, where f stands for 'focus', then the other non-f tests are ignored. 'x' is the reverse, meaning hard-skip the test. This is fine as a temporary fix, but you don't wan't to commit it to git or forget it was there or you end up with forgotten tests. (https://jasmine.github.io/api/2.8/global.html#fdescribe)

Ultimately a proposed solution that helped me find a way forward was to edit the `test.ts` file. (https://stackoverflow.com/a/50636750 - this os one of those obvious-in-hindsight things that makes it hard to google for) By itself this is just as non-useful as the advice to use `fdescribe.` . However, along with some webpack magic I was able to come up with a sane solution.

## Solution



Talk about how to customise webpack


--
https://stackoverflow.com/questions/50969709/how-to-get-karma-to-run-tests-of-a-specific-angular-6-component-library-inside-w
https://stackoverflow.com/questions/50333461/karmawebpackmocha-running-single-tests
https://dzone.com/articles/running-karma-test-case-for-single-spec-file
https://medium.com/@bebraw/running-individual-tests-with-karma-mocha-89aece8ba18b
https://stackoverflow.com/questions/29150998/karma-running-a-single-test-file-from-command-line
https://stackoverflow.com/questions/48637506/how-to-exclude-folder-components-for-unit-testing-in-angular-4-using-karma-con
https://github.com/angular/angular-cli/issues/10670
https://github.com/angular/angular-cli/issues/9949


---

[^1]: [https://stackoverflow.com/questions/40683673/how-to-execute-only-one-test-spec-with-angular-cli](https://stackoverflow.com/questions/40683673/how-to-execute-only-one-test-spec-with-angular-cli)

[^2]: [https://github.com/angular/angular-cli/issues/9949](https://github.com/angular/angular-cli/issues/9949)

  [^3]: [https://github.com/angular/angular-cli/issues/10670](https://github.com/angular/angular-cli/issues/10670)

