---
layout: post
title:  "Keep your reviewers happy by using Git rebase!"
date:   2023-11-05 00:00:00 +0000
categories: git
---

## Introduction

In today's dynamic software engineering landscape, Git reigns supreme as the de facto choice for version control. This powerful version control system has become the cornerstone of modern development, facilitating seamless collaboration, tracking changes, and ensuring code stability (thank you [Linus](https://en.wikipedia.org/wiki/Linus_Torvalds)!). Understanding how to properly work with Git can significantly enhance a developer's ability to collaborate effectively with their team. In this article we'll talk a bit about the advantages of using Git rebase instead of Git merge when working in a collaborative environment.

## Working as a team

Most of the meaningful work that we

Usually in these environments developers create feature branches in which they develop their solutions and open a pull request to the main branch whenever that feature is ready. Then the reviewers will have to go through the changes of this PR to validate the work before merge it to the master branch.

In a (hopefully!) typical scenario the developments done have no conflicts with the master branch so the feature branch can be seamlessly merged into the master branch after being reviewed.

However there are times in which conflict arises between the master branch and the feature branch. This means that the changes introduced in the feature branch are not compatible with the changes done in the master branch after the creation of the feature branch. At this point merging the feature branch into the master branch is not allowed until these conflicts have been resolved. To resolve them we have two choices:

  - merge the changes from the master branch into the feature branch and resolve the conflicts there

  - rebase the feature branch into the master branch

## Using Git merge

When this happens many developers often use git merge to fix the conflicts. This is the easiest solution because


## Using Git rebase

A better solution is to use git rebase. With this option we simply move the commits of feature branch into the top of the master branch


## Other uses for git rebase

## Conclusion
