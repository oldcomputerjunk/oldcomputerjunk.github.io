---
layout: post
title:  "Welcome to Jekyll! (Switching over from Wordpress)"
date:   2016-04-04 19:46:38
categories:
- blog
tags:
- blog
---

I finally decided to migrate my blog over to work with GitHub pages.
I haven't published much recently in part because it was always a bit tedious 
doing code-example heavy posts, also I really like the idea of a statically
served site with no SQL database underneath, you get backup for free
via git clone and can use branches and other tricks as needed.

Editing locally, mucking around and testing before publishing is also
often a _lot_ quicker than waiting for Aussie rural-strength Internet to catch up ...

The other cool thing is this can be self hosted easily if required.
I did all my testing using Docker on my local machine.

### Jekyll publishing

So my blog now uses a framework called [Jekyll](http://jekyllrb.com).
Jekyll works by compiling the blog, the articles for which are now
written using the [MarkDown](https://daringfireball.net/projects/markdown/basics) language,
into a set of static pages whilst still giving the
appearance of a dynamic web site. So no need to worry about Wordpress security
patches anymore.

I basically just followed some tutorials including [this one](https://medium.com/@katfukui/the-design-portfolio-workflow-a94030d0b39e#.wdaxjubi0) and 
and [this one](http://fettblog.eu/blog/2014/01/02/how-to-switch-from-wordpress-to-jekyll/) 
and used this [docker container](https://github.com/jekyll/docker) although
having never used Ruby in any serious way and being completely new to Jekyll
I got stuck for a little while trying to make it actually serve without a "Permission Error."

_**It turns out that with Jekyll if you don't have an "index.html"
you get a counter-intuitive "Forbidden" error page...**_

I found the instructions for the Docker container were not completely clear,
in the end I had to use what felt like a trick:

1. Generate the new site whilst mapped to one directory on my host, then
2. Actually run the container mapping to the just generated directory

This is likely to do with the fact that everything runs from `/srv/jekyll` in the container
including both site creation operations and serving, but we want to serve from
the created directory which is nested... confused much?

The commands I used for testing under docker were:

{% highlight bash %}
docker pull jekyll/jekyll:pages
docker pull jekyll/jekyll:builder
mkdir -p srv/site1
cd srv
docker run --rm --label=jekyll --volume=$(pwd):/srv/jekyll \
  -it -p 127.0.0.1:4000:4000 jekyll/jekyll:pages jekyll new site2
docker run --rm --label=jekyll --name=blog-test \
  --volume=$(pwd)/site2:/srv/jekyll \
  -it -p 127.0.0.1:4000:4000 jekyll/jekyll:pages
{% endhighlight %}

Here, you end up with the site in ```site2``` on the host. The first command creates it; the second serves from 'inside' it...
This is all only an issue for testing locally anyway - 
once I get stuff in the right place on my github repository things should be sorted.

### Wordpress import

I migrated all the Wordpress entries over using a tool called [Exitwp](https://github.com/thomasf/exitwp).
This was fairly easy, just export from Wordpress, place the XML file
in the right place and run a python script.  This then generates a directory structure that you 
copy over the freshly created jekyll blog. To fix the permalinks to match exactly I had then configured with 
Wordpress, I simply had to add `permalink: /blog/:year/:month/:day/:title/` to the Jekyll `_config.yml` file.

Out of the box the style is a bit sparse, so the next step was to add a style.

For now I have started with one called [lanyon](https://github.com/poole/lanyon).
I had to tweak it a bit, by default it needed a Ruby gem that wasn't in my
Docker image and I wasn't interested in modifying it at this point in time.

I also wanted to show the full latest article on the home page, so I
duplicated the old home to an "[Posts Archive](/allposts.html/)" page.

You can see the menu tucked away to the left of the page now.

### Things still to do

I need to work through a number of things, but I'll do that gradually over the next while.

* Fix broken syntax highlighting. I was using the (very good) Crayon syntax highlighter but that hasn't converted properly.
* Fix broken pictures in some cases
* Create a tag cloud - I liked that :-)
* Create a sitemap XML
* Check everything...

Missing features at this stage include tag cloud, month by month archive - probably wont bother,
blogroll - probably wont bother, unless asked - I'll probably try and work out how to link back
to [Planet Linux Australia](http://planet.linux.org.au/), and comments - again, wont bother
although I might improve the contact form. I may experiment with colours / styles as well.

Also I have a few interesting things lined up to blog, so that will be a good test. Hopefully soon...
