---
layout: post
slug: long-time-no-blog
title:  "Well that has been a while"
date:   2020-07-11 01:00:00
categories:
- blog
tags:
- wtf2020
- github
- blog
---

I know I'd been preoccupied with other things much more than I'd have liked, but I just discovered that its been over a year since I blogged :-(

Still, now is as good a time as any to try and rectify that!

Objectives:
- not to let perfect be the enemy of good
- even if it is just something simple, it might help someone

Probable topics going forward:
- hacking embedded things
- car hacking?
- information security learnings
- maybe even gardening!

---

This was almost dead and buried before it started unfortunately.

I made this new entry, pushed it, waited for github pages to build, and crickets.  The last commit has a red 'X' next to it --?

Click on the red "X" and you get:

<img src="/images/Screenshot-at-Jul-11-20-23-35.png" alt="Screenshot of error detail" class="inline"/>

Clicking on 'details' only says 'GitHub Pages failed to build your site.'  Seriously GitHub, how about a more useful error message? There is a link, "View more details on GitHub Pages" which only goes to [online help](https://pages.github.com/). Gee, thanks.

I'm very rusty on this, so I started by poking around the various repository menus. I'm guessing lots has changed in the past year or two. As my memory comes back and as I read through the online help, my first suspect is the `_config.yml` file. Lets find out.

According to (Troubleshooting Build Errors](https://docs.github.com/en/github/working-with-github-pages/troubleshooting-jekyll-build-errors-for-github-pages-sites#troubleshooting-build-errors) it could be one of the following:
- unsupported plugins (quite possible)
- respository size limit (unlikely)
- modified `source` setting in `_config.yml` - don't even have that?
- a filename has a colon character which is unsupported. It shouldn't be this given things were working...

The help also lists some common `_config.yml` problems:
- ensure spaces are used (✅)
- spaces after colons in key value pairs (✅)
- use only UTF-8 (✅)
- quote special characters (✅)
- Use `|` and `>` in multi-line. hmm. 🤔

There is a multiline value for `description` with no bar character. Lets check this in the [YAML validator](https://codebeautify.org/yaml-validator) suggested.  Yep, valid YAML. So not that.

Further detail - config file:
```
# Site settings
timezone: Australia/Adelaide
title: Just a pile of Old Computer Junk
tagline: '"Its life Jim, but not as we know it"'
email: blog@oldcomputerjunk.net
description: > # this means to ignore newlines until "baseurl:"
  This is my personal blog about various software development, devops,
  information security, networking & embedded electronics topics.
baseurl: "" # the subpath of your site, e.g. /blog/
url: "http://blog.oldcomputerjunk.net" # the base hostname & protocol for your site
twitter_username: pastcompute
github_username:  oldcomputerjunk
pygments: true
permalink: /:year/:title/

paginate_path: '/blog/page:num/'
paginate: 1

# Build settings
markdown: kramdown

kramdown:
  auto_ids: true
```

So, `kramdown` is some kind of plugin perhaps. As is `pygments`? time to read the manual further ... 🕟

There are a whole bunch of other troubleshooting tips, but couple to an error message (which I don't have!).  Not having _any_ error message (in theory I should) might point to something much more fundamental, but this is really frustrating. At this stage I'm faced with building Jekyll locally, which was a bit tedious last time, I really CBF firing up docker at this point. I actually want to write a _different_ blog entry :-(

As a last resort, I revert back to the last successful commit by branching off my new work and then reverting. Go git push...

Well, this time the build worked. So something is broken in this blog entry.

First, in this same update I fixed some broken formatting in [/reminder/](). So lets merge that back to master and try again ... 🕟 ... well, that worked.

So what is wrong with my new article? Only thing obvious is the filename - I have the date transposed, so fix that, but that should not be it - the filename is really arbitrary only used for sorting, the date comes from the blog entry YAML metadata header. Well, try again ... 🕟 ...

Failed again! Time to rip things out and divide and conquer.

I started by copying another article, editing the metadata then adding text.

I had added a tag `2020` so first change that to `wtf2020` maybe it doesnt like tags with a number ... 🕟 ...
