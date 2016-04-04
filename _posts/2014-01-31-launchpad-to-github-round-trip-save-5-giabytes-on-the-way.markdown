---
author: admin1
comments: false
date: 2014-01-31 11:52:39+00:00
layout: post
slug: launchpad-to-github-round-trip-save-5-giabytes-on-the-way
title: Launchpad to Github round trip, save 5 giabytes on the way
wordpress_id: 475
categories:
- oqgraph
tags:
- git
- github
- oqgraph
- sql
---

As alluded to in my [previous entry](http://blog.oldcomputerjunk.net/2014/oqgraph-bazaar-adventures-in-migrating-to-git/), as part of working on the OQGraph storage engine I have set up a development tree on GitHub, with beta and released code remaining hosted on Launchpad using bzr as part of the MariaDB official codebase.  This arrangement is subject to the following constraints:



	
  * I will be doing most new development via Github

	
  * I will regularly need to be integrate changes from Github back to Launchpad

	
  * I occasionally may need to forward port changes from Launchpad to Github, these may arises from changes in MariaDB itself.

	
  * The codebase on GitHub consists solely of the storage engine without the overhead of the rest of MariaDB.

	
  * This provides a much more accessible route to development or testing of the latest OQGraph code - downloading just the source tree of MariaDB, and replacing the released OQGraph directory with the development code, rather than having to bzr branch or git clone all of MariaDB.


Until I have gone around the loop a couple of times, I expect some glitches in the process to arise and need resolving.

Some resulting branches in GitHub:

	
  * **baseline** : this branch locates where we started from in bzr

	
  * **master** : this branch is the result of merges from launchpad-tracking with ongoing development via github

	
  * other branches such as **maria-mdev-xxx** and **maria-launchpad-xxx** for work on specific bugs or features


But things had to start somewhere, so here is how I extracted the storage engine code from MariaDB whilst retaining full past history.


## First things first - cloning MariaDB from launchpad


Since sometime around 1.8 a new remote helper was made available with git, for cloning from and pushing to bzr repositories with a local git repository.  On Debian Wheezy this is provided with the package **git-bzr**.  So the first step is to clone the mainline MariaDB repository into a git repository.  It turns out you need multiple gigabytes of local storage for this; the resulting .git directory is 5.7 gigabytes.  It also takes rather a long time, even over a 12 Mbps link - the bottleneck is not the network, but seems to be **bzr** itself.

Prior to cloning to git, however, I ensured I also had a tracking branch created inside Launchpad, to facilitate upstream integration.  First I followed the instructions on the MariaDB wiki [1] to create a local bzr repository, and created my maintenance branch from **lp:maria**.  The end result should look like this:

[crayon lang="bash"].../bzr/oqgraph-maintenance$ bzr info
Repository tree (format: 2a)
Location:
shared repository: /scratch/bzr
repository branch: .

Related branches:
push branch: bzr+ssh://bazaar.launchpad.net/~andymc73/maria/oqgraph-maintenance/
parent branch: bzr+ssh://bazaar.launchpad.net/+branch/maria/
[/crayon]

Now it is possible to make a git repository from the bzr repository.

[crayon lang="bash"]cd ../..
mkdir git && cd git
git clone bzr::../bzr/oqgraph-maintenance
[/crayon]

MariaDB is showing its long and venerable history, with over 80000 commits in a very complex branch structure; the downside of this includes significant lag on simple operations such as **git status** or firing up the GUI **gitg**.  But otherwise, the history is complete and corresponds with that within the bzr repository; the gir-bzr remote helper does its job well.

We also have remotes that actually point to the bzr repository:

[crayon lang="bash"]$ .../git/oqgraph-maintenance$ git remote -v
origin    bzr::/scratch/bzr/maria-oqgraph-maintenance/ (fetch)
origin    bzr::/scratch/bzr/maria-oqgraph-maintenance/ (push)
[/crayon]

One thing I had to do, was cleanup an extraneous tag **HEAD** which I removed the ref for it to avoid ambiguous ref errors (this doesn't interfere with round trip operations):
[crayon lang="bash"]git tag -d HEAD
git reset --hard HEAD[/crayon]




### Lossless Data Reduction


Now at this point I could have create a new repository on Github and pushed the entire clone to it, and be done with it.  But thats a lot of data, clogging up both Github and my connections.  And not much use if I wanted to do a clone for some quick hacking in a spare half hour waiting for a plane or something - see bullet five in the introduction above.

One way to do this is to use **git filter-branch** with a subdirectory filter filter.  However, some of the OQGraph code for a lot of its history existed in two unrelated directories, and the past history of the files in the second directory would become lost.  Specifically, the test harnesses for the storage engine previously existed under **mysql-test/suite/oqgraph** and now live under **storage/oqgraph/mysql-test**. The trick is to rewrite history with a 'fake' directory so that the files previously under **mysql-test/suite/oqgraph** now appear to have resided under **storage/oqgraph/maria/mysql-test/suite/oqgraph**.  This preserves not only past changes in those files, but even the move to the final location, through the eventual subdirectory filter.  Note, at this point we are not ready to apply a subdirectory filter as yet.

The subdirectory filter operation takes a very long time, so we also look for a way to quickly remove a lot of code.  The history goes back to 2000 and OQGraph entered the scene in about 2010. So we can run a different filter-branch operation to remove all commits older than that point.  However, because we are preserving the merge history, the default method for doing that still leaves behind a lot of cruft back to 2007. So we need to apply yet another filter tor remove the empty merge commits.

Finally we can apply a subdirectory filter.


### Putting it all together - remove old commits and free up space


I started by making a new clone.  We need to retain the original for round trip integration, and for starting again if we made a mistake.  Having done that, we want to find the oldest commit in any part of the OQGraph software.

[crayon lang="bash"]cd ..
git clone oqgraph-maintenance oqgraph-standalone
cd oqgraph-standalone
git log --all --format="%H %ai %ci %s" mysql-test/suite/oqgraph
git log --all --format="%H %ai %ci %s" storage/oqgraph[/crayon]

From a visual examination of the log output, the oldest relevant commit might be for example _7de3c2e1d02a2a4b645002c9cebdaa673f05e767_.  It would be feasible to write a script to work this out by parsing and sorting the dates, but I didn't bother spending the time at this point.  To remove the old commits, we need to make a new 'start of history', which we do by adding a graft then running a filter branch command.  We could run just **filter-branch** by itself, but at the same time we also remove everything that is not part of the OQGraph storage engine.

(Note that the each command one long line, I have separated each line by a blank here)
[crayon lang="bash" wrap="true"]git rev-parse --verify 7de3c2e1d02a2a4b645002c9cebdaa673f05e767 >> .git/info/grafts

git filter-branch --index-filter 'git rm --cached -qr --ignore-unmatch -- . && git reset -q $GIT_COMMIT -- storage/oqgraph mysql-test/suite/oqgraph' --tag-name-filter cat --prune-empty -- --all

git show-ref |awk '{print $2}' |egrep ^refs/original|xargs -L1 git update-ref -d

git reflog expire --expire=now --all

git gc --aggressive --prune=now[/crayon]

Here, the **index-filter option** manipulates the git index, one entry at a time; we unstage the current commit, and re-commit just the files of interest.  This is faster than a subdirectory filter because we don't check out the entire tree, and also we can keep two directories instead of just one.  The **tag-name-filter** is crucial, without it we loose all the MariaDB history tags. Having **prune-empty** removes commits with no data, and **all** ensures we look at all branches rather than just the direct ancestors of HEAD.

This is followed up by space reclamation: in three steps, we remove backup refs created by filter-branch, prune all obsolete refs, then garbage collect to reclaim space.

Although faster than a subdirectory filter it still takes a couple of hours to run, but the results are amazing: the entire repository has shrunk to just 7 Megabytes!


#### Cleanup cruft and  normalising the directory structure


There is still some work to do however; a large number of merge commits remain, all of which are empty, comprising history both beyond and after the graft point, and obscuring our OQGraph commits.  The previous argument **prune** doesn't remove merge commits, so we need to do that differently.   The data is dramatically reduced, so subdirectory filter will be quite fast, and also removes merge commits, but we still need to retain history out in **mysql-test/suite/oqgraph** which would be lost if we just just did a **subdirectory-filter**.  So we need to rewrite the related paths first.

(Note again each command a one long line, I have separated lines by blanks)
[crayon lang="bash" wrap="true"]git filter-branch --index-filter 'git ls-files -s | sed -e "s@mysql-test/suite/oqgraph@storage/oqgraph/maria/mysql-test/suite/oqgraph@" | GIT_INDEX_FILE="$GIT_INDEX_FILE.new" git update-index --index-info && test -e "$GIT_INDEX_FILE.new" && mv "$GIT_INDEX_FILE.new" "$GIT_INDEX_FILE" || true' --tag-name-filter cat --prune-empty -- --all

git filter-branch -f --subdirectory-filter storage/oqgraph --tag-name-filter cat --prune-empty -- --all[/crayon]

Here, running **index-filter** with **git update-index** we can 'rename' file paths affected by a commit without aggecting the actual commit.  So I used **sed** to change **mysql-test/suite/oqgraph** paths to **storage/oqgraph/maria/mysql-test/suite/oqgraph**.  These now survive the subdirectory-filter and we retain a complete history of OQGraph files without really changing anything.

I ran one final clean up:
[crayon lang="bash" wrap="true"]git show-ref |awk '{print $2}' |egrep ^refs/original|xargs -L1 git update-ref -d
git reflog expire --expire=now --all
git gc --aggressive --prune=now
git repack -a -d -l[/crayon]

Now I had a complete history of OQGraph, with only OQGraph, including contemporary MariaDB tags, which I pushed to Github.
The results are at [https://github.com/andymc73/oqgraph](https://github.com/andymc73/oqgraph), and you could find the corresponding commits in the MariaDB trunk on Launchpad if you desired. 

The draft procedure for round-tripping back to Launchpad is described in the Github [maintainer-docs](https://github.com/andymc73/oqgraph/blob/master/maintainer-docs/Synchronising_with_Launchpad.md), this is still a work in progress. 

--

[1] [https://mariadb.com/kb/en/getting-the-mariadb-source-code/](https://mariadb.com/kb/en/getting-the-mariadb-source-code/)

