---
layout: post
slug: angular7-esoterica-fetch
title:  "Package updates don't always help (bug: unsupported BodyInit type)"
date:   2019-03-31 01:00:00
categories:
- tech
tags:
- angular
- angular 7
- ionic4
- ionic
---


I had an issue with Ionic4 / Angular7 where I encountered the following error:

```
Uncaught Error: unsupported BodyInit type
    at Response.Body._initBody (...\fetch.js:231)
    at new Response (...\fetch.js:390)
    at XMLHttpRequest.xhr.onload (...\fetch.js:437)
    ...
```

Essentially I had been monkey-patching `XMLHttpRequest` to try and cache some queries my app was making (long story, for some other time.)   This had been working for a good year or more, then one day in Safari the desired behaviour just broke. For some reason it didn't happen in Chrome or Firefox but thats a problem for another day as well.

Digging into the detail eventually revealed the above console error message.

Skipping a lot of boring and frustrating debugging and detail, I eventually discovered that the Angular `zone.js` component patches the global `fetch` (which might not even be present in some browsers), by way of `polyfills.ts` which is a default file that all Angular apps get when scaffolded.

It turns out you can turn this off, and this was documented in the polyfills file:

```
/**
 * By default, zone.js will patch all possible macroTask and DomEvents
 * user can disable parts of macroTask/DomEvents patch by setting following flags
 */
```

So I dutifully made the change:
```
(window as any).__Zone_disable_fetch = true;
```

...and it didn't work. Sigh.

Some debugging later (meaning, copious splattering of `console.log` in the output of `npx ng build`) I discovered something I'd never realised about typescript modules before, which is that code before the initial `import 'blah'` is actually executed _after_ the code in those imports is first executed.

Or strictly speaking, the javascript output by the compiler, appears to 'hoist' the import-generated code above all other code in the typescript file in the generated output. (Hoist in javascript originally means, that all variables declared with var, are actually initialised together at the start of execution of the script, a bit like how in C/C++ global static variables are initialised before code is run)

Or, as suggested in the bug report (see below) it is webpack that moves things around. Either way, as documented it was never going to work.

This seemed at odds with the comment, so I fell down another rabbit hole and ended up in the `angular-cli` code in Github:
```
 * By default, zone.js will patch all possible macroTask and DomEvents
 * user can disable parts of macroTask/DomEvents patch by setting following flags
 * because those flags need to be set before `zone.js` being loaded, and webpack
 * will put import in the top of bundle, so user need to create a separate file
 * in this directory (for example: zone-flags.ts), and put the following flags
 * into that file, and then add the following code before importing zone.js.
 * import './zone-flags.ts';
```

See:
- (https://github.com/angular/angular-cli/blob/9aefe8371c9b1d4f3314f950c4e52df4231c3545/packages/schematics/angular/application/files/src/polyfills.ts.template#L31)
- (https://github.com/angular/angular-cli/commit/c83be5e555dbad53ad445ef0c3cacaf4ac5730dc)

After this, I discovered a bug report about the problem I'd found: (https://github.com/angular/angular-cli/issues/12886)

Of course, the project I was working with was first scaffolded some time ago, and since then others had been down my path and updated the documentation, as per the above links. Of course, because the comment is in a scaffolded file, simply having the latest version pulled down by npm cannot update those files which are source code in an existing project.

This was all a bit tedious, and unfortunately I don't really have a constructive solution. In an extreme case a super-smart CLI might scan your code and report differences, but it would need to first pattern match against all history of all scaffolded files that still exist so that it could exclude your own modifications. Seems a bit extreme and difficult.
Perhaps comments in scaffolding files could cross-reference to the git history (it is not always easy to find where scaffolded files come from) to give debugging developers a hint.
