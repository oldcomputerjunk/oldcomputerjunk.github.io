---
author: admin1
comments: false
date: 2013-09-24 11:21:53+00:00
layout: post
slug: some-funky-stuff-with-git-working-with-tags
title: Some funky stuff with git - working with tags
wordpress_id: 431
categories:
- howto
tags:
- git
- github
---

Tags form an important part of any version control system. One of the most common use cases is to mark a revision that corresponds to a released software version; this allows reconciliation of bug reports to the correct code, for example.

There are a few tag related operations that are not immediately obvious in git however.


### Listing all commits between two tags.


This is simple enough:
```bash
git log START^..ENDPOINT --oneline```


### Listing all tags between two revisions.


The following command will list all tags from and including ref OLDEST_TAG_OR_BRANCH, along with the abbreviated commit SHA and brief log.  Omit the caret (^) to exclude the starting point from the list.  HEAD means go until the most recent commit in this line, append a caret to exclude it if it is tagged, or replace HEAD with another tag or branchpoint.
```bash
git log OLDEST_TAG_OR_BRANCH^..HEAD --oneline \
        --decorate --simplify-by-decoration | grep '(tag:'```
git log, and its cousin git rev-list have quite a lot of formatting and selection options, so checkout the man pages (i.e. man git-log and man git-rev-list ; note, individual git subcommand man pages are accessed by prepending 'git-' to the subommand.)


### Preserving tags during a filter-branch operation.


The command git filter-branch is a powerful tool that can be used to do things such as change the email address of a user of every commit they made, remove specific words from a commit message, fix common spelling mistakes, rearrange the file tree, etc.  It should not be use ad-hoc as it does effectively rewrite the history, invalidating  working copies so should be use with care.  It is often used when migrating from another system into git, or migrating servers, etc.

One trap with filter-branch is that it is easy to lose tags when using the command.  Filter-branch actually makes a new branch, leaving the original untouched in the repository as a kind of backup, called 'refs/original/refs/heads/YOUR_BRANCH.  Tags remain attached to this _original_ branch, and are not copied to the new branch unless the --tag-filter is employed concurrently with whichever filter is being used.  Then when you use or clone the replacement YOUR_BRANCH, the tags are "gone"!  This is easily fixed:

(a) Edit every commit message in current branch, removing the word "frooble" and replacing it with "foo".  This happens to "forget" tags on the way
```bash
$ git tag
tag1
tag2
$ git filter-branch --msg-filter "cat - | sed 's/frooble/foo/g'
$ git tag       # <-- trap - shows _all_ tags in repo, not just your branch
tag1
tag2
$ git tag HEAD  # <-- easy to forget HEAD, so you dont find problem until later
# no output!```
(b) Edit every commit message in current branch, removing the word "frooble" and replacing it with "foo", this time making sure the tags move to the branch.
```bash
$ git filter-branch \
        --msg-filter "cat - | sed 's/frooble/foo/g' \
        --tag-name-filter cat
$ git tag HEAD 
tag1
tag2```


### Pushing your tags to your remote.


Remember the --tags flag, so that your tags go up to github, etc:
```bash
$ git push --tags origin```


### Removing tags on remote.


```bash
$ git tag -d sometag
$ git push origin :sometag```
