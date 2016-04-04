---
author: admin1
comments: false
date: 2014-01-27 13:01:00+00:00
layout: post
slug: oqgraph-bazaar-adventures-in-migrating-to-git
title: OQGraph - bazaar adventures in migrating to git
wordpress_id: 469
categories:
- oqgraph
tags:
- git
- github
- LCA2014
- linux
- oqgraph
- sql
---

## Background


I have been acting as a 'community maintainer' for a project called OQGraph for a bit over a year now.

OQGraph provides graph traversal algorithms over SQL data for MariaDB.  Briefly, it is very difficult if not impossible to write a SQL query to find the shortest path through a graph where the edges are fairly typically represented as two columns of vertex identifers.  OQGraph as of v3 provides a virtual table that allows such an operation via a single SQL query.  This thus provides a simple mechanism for existing data to be queried in new and novel ways.  OQGraph v3 is now merged into MariaDB 10.0.7 as of 16 December, 2013.

_**Aside:** I did a talk [1][2][3] about this subject at the Linux.conf.au OpenProgramming miniconf. I really didn't do a very good job, especially compared with my SysAdmin [4][5][6] miniconf talk; I lost my place, "ummed" way too much, etc., although audience members kindly seemed to ignore this when I talked to them later :-) I know I was underprepared, and in hindsight I tried to cover way too much ground in the time available which resulted in a not-really coherent story arc; I should have focused on MTR for the majority of the talk. But I digress..._


_**Correction**: I also had a snafu in my slides; OQGraph v3 supports theoretically any underling storage engine, the unit test coverage is currently only over MyISAM but we plan to extend it to test the other major storage engines in the next while._


## Launchpad and BZR


MariaDB is maintained on [Launchpad](https://launchpad.net/maria). Launchpad uses bazaar (_bzr)_ for version control.  Bazaar already has a reputation for poor performance, and my own experience using it with MariaDB backs this up.  It doesn't help that the MariaDB history is a long one, the history converted to git shows 87000 commits and the .git directory weighs in at nearly 6 GBytes!  The MariaDB team is considering migration [7] to github, but in the meantime I needed a way to work on OQGraph using git to save my productivity, as I am only working on the project in my spare time.


## Github


So heres what I wanted to achieve:



	
  1. Maintain the 'bleeding edge' development of OQGraph on Github

	
  2. Bring the entire history of all OQGraph v3 development across to Github, including any MariaDB tags

	
  3. Maintain the code on Github without all of the entirety of MariaDB.

	
  4. Be able to push changes from github back to Launchpad+bzr


Items 1 & 3 will give me a productivity boost.  The resulting OQGraph-only repository with entire history almost fits on a 3½inch floppy! Item 1 may help make the project more accessible in the future.  Item 2 will of course allow me to go back in time and see what happened in the past.  Item 3 also has other advantages: it may make it easier to backport OQGraph to other versions of MariaDB if it becomes useful to do so.  And item 4 is critical, as for bug fixes and features to be accepted for merging into MariaDB in the short term it is still easiest to maintain a bzr branch on launchpad.

To this end, I first created a maintenance branch on Launchpad: [https://code.launchpad.net/~andymc73/maria/oqgraph-maintenance](https://code.launchpad.net/~andymc73/maria/oqgraph-maintenance). I will regularly merge the latest MariaDB trunk into this branch, along with completed changes (tested bugfixes, etc.) from the github repository, for final testing before proposing for merging into MariaDB trunk.

Then I created a standalone git repository.  The OQGraph code is self contained in a subdirectory of MariaDB, storage/oqgraph . The result should be a git repository where the original content of storage/oqgraph is the root directory, but with the history preserved.

Doing this was a bit of a journey and really tested out by git skills, and my workstation!  I will describe this in more detail in a subsequent blog entry.

The resulting repository I finally pushed up to github and can be found at [https://github.com/andymc73/oqgraph](https://github.com/andymc73/oqgraph). I also determined the procedure for merging changes back to Launchpad, see the file [Synchronising_with_Launchpad.md](https://github.com/andymc73/oqgraph/blob/master/github/Synchronising_with_Launchpad.md)



[1] [https://lca2014.linux.org.au/wiki/index.php?title=Miniconfs/Open_Programming#Developing_OQGRAPH:_a_tool_for_graph_based_traversal_of_SQL_data_in_MariaDB](https://lca2014.linux.org.au/wiki/index.php?title=Miniconfs/Open_Programming#Developing_OQGRAPH:_a_tool_for_graph_based_traversal_of_SQL_data_in_MariaDB)

[2] Video: [http://mirror.linux.org.au/linux.conf.au/2014/Tuesday/139-Developing_OQGRAPH_a_tool_for_graph_based_traversal_of_SQL_data_in_MariaDB_-_Andrew_McDonnell.mp4](http://mirror.linux.org.au/linux.conf.au/2014/Tuesday/139-Developing_OQGRAPH_a_tool_for_graph_based_traversal_of_SQL_data_in_MariaDB_-_Andrew_McDonnell.mp4)

[3] Slides: [http://andrewmcdonnell.net/slides/lca2014_oqgraph_talk.pdf](http://andrewmcdonnell.net/slides/lca2014_oqgraph_talk.pdf)

[4] [http://sysadmin.miniconf.org/presentations14.html#AndrewMcDonnell](http://sysadmin.miniconf.org/presentations14.html#AndrewMcDonnell)

[5] Video: [http://mirror.linux.org.au/linux.conf.au/2014/Monday/167-Custom_equipment_monitoring_with_OpenWRT_and_Carambola_-_Andrew_McDonnell.mp4](http://mirror.linux.org.au/linux.conf.au/2014/Monday/167-Custom_equipment_monitoring_with_OpenWRT_and_Carambola_-_Andrew_McDonnell.mp4)

[6] Slides: [http://andrewmcdonnell.net/slides/lca2014_sysadmin_talk.pdf](http://andrewmcdonnell.net/slides/lca2014_sysadmin_talk.pdf)

[7] [https://mariadb.atlassian.net/browse/MDEV-5240](https://mariadb.atlassian.net/browse/MDEV-5240)
