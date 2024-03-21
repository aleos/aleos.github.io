---
title: Mercurial Installation
date: 2011-12-18T12:52:00Z
tags: [mercurial, server]
---

Mercurial is a distributed version control system (SCM). It stands out for its ease of use and, consequently, the short time it takes for newcomers to get acquainted with it. At the same time, Mercurial is sufficiently functional for the vast majority of tasks.

In this article, I will describe how to create what I consider to be a fairly standard configuration that will meet the needs of most small development groups. In general, installing Mercurial is not complicated, but it can pose some difficulties for the uninitiated.

Although distributed SCMs do not require a central repository, it is often very useful to have an "integration" repository as the place from where one can always obtain the current state of development from all developers, and as a place for exchanging change sets.

So, the integration repository will be stored on a Linux server, to which access is available via SSH. An advantage of this configuration is also the possibility for remote developers to access the repository if the server has internet access.

To start with, for security purposes, you need to create a user group in `/etc/groups`, containing all developers who should have access to the repository. For example, this group will be called `scm`. See your distribution's documentation for how to create a group and add users to it. Our repository will be located in the `/hgrepo` directory. Let's create a new repository and restrict access to it to only those in the `scm` group:

```shell
$ cd /hgrepo
$ hg init myrepo
```

The repository is created and almost ready for use. Let's set the repository access rights:

```shell
# chgrp -R scm myrepo # set the group that has access to the repository
# chmod -R o= myrepo # deny access to anyone not in the scm group
# chmod -R g+w myrepo # grant write access to the repository for the group
# find myrepo -type d -exec chmod g+s {} ; # all new files and directories will have the scm group
```

For convenience, let's create a symbolic link at the root:

```shell
$ cd /
# ln -s /hgrepo/myrepo
```

Everything is ready. To use the repository, developers must have Mercurial installed, with a version at least as new as that on the server, and have their username specified in the `.hgrc` file in their home directory. You can check the Mercurial version with the command:

```shell
$ hg --version
```

To make a clone, execute:

```shell
$ hg clone ssh://<username>@<servername>//hgrepo
```

Note the double slash before the repository name. This indicates that the path is specified relative to the root of the file system, not relative to the user's home directory, as is the default.

## Links:

- [http://ru.wikipedia.org/wiki/Mercurial](http://ru.wikipedia.org/wiki/Mercurial)
- [http://mercurial.selenic.com/wiki/MultipleCommitters](http://mercurial.selenic.com/wiki/MultipleCommitters)
