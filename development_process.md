# Development workflow:

## Overview

This document is intended to capture the process involved on making contributions to Jiocloud.
It is divided into three sections:

* *common* - Covers topics common to both code as well as configuration submissions.
* *puppet-rjil* - Covers change process for puppet-rjil. Provides description of how
deployments and upgrades of puppet-rjil are managed.
* *source code changes* - Explains the process for getting source code packaged and into production.

## Common tasks

All code changes are added as commit messages in git and submitted as pull requests in github.
Changes that need to be submitted upstream may also need to be submitted to Gerrit.
A firm understanding of Github and Git is required for anyone who is going to submit code.
Users should at least know how to login to jenkins and check their build state.
Submit changes upstream (at least for openstack and contrail) may also require an understanding of Gerrit.

At a minimum, everyone needs to understand how to:
* git: perform the basic git operations: add, commit, rebase, cherry-pick, push.
* github: fork repos, create, update pull requests, and comment on pull requests.
* gerrit: use git review, track changes through gui.

All deployments and updates of jiocloud are managed as code changes in
[puppet-rjil](https://github.com/jiocloud/puppet-rjil).
This repository serves as both the instructions for what our current production cloud should look like,
and as the collaboration platform through which most changes will be submitted and peer reviewed.
For new developers, it will also serve as the process through which they will be trained.

This section is intended to explain how to get changes merged into this repository as well as how those changes will work their way to production. It is not intended to serve as an explanation of how that repository works, or how code should be written for that repository. The repository has it's own [README](https://github.com/JioCloud/puppet-rjil/blob/master/README.md) with that information.

Until we get better cadence for getting code merged into upstream and released, most of the patches that will land in the production environment will be introduced through this repository.

### Getting Changes to Production

All changes related to puppet-rjil go through a multi-step process to work their way to production.

![Dev Process](/images/puppet-rjil-dev-process.png)

The above diagrams shows the steps required to get things merged as well as the roles involved in merging code.

#### Writing patches:

The first step in the process is that a developer should write a patch. The patch may or may not have an associated ticket (which updates require tickets and which ones don't is considered as out of the scope of this document).

A patch in this context is a single unit of change. It is a set of code that is all intended to support a single change to the overall behavior of the system. During the review process, patches may get rejected if they are not focused on a single feature or bug. In these cases, reviewers should indicate that the developers should split up their patch into multiple patches.

##### Commits:

Patches are organized as one or more commits. Commits are unit of code changes as described by git. In generally, patches should only contain a single commit. In some cases for more complicated patches, it is possible that multiple commits make sense.

##### Commit Messages:

Each commit is labeled with a commit message. The commit message provides supporting information to better understand the context of a specific code change. As a general rule, the commit message should contain enough context for a reviewer to completely understand and review code without any additional questions to the developer about the intention, and potential effects of the change.

All commits should be structured as follows:

a commit title - a short (less than 50 characters) description of the patch
a blank line
additional information for each change

In general, commit messages should contain the following:
* a description of the change
* motivation for the change
* previous behavior, along with why that behavior was problematic
* the new behavior
* any links to relevant tickets, blueprints, or other supporting material

These requirements might seem extreme, but remember, the primary purpose of a commit message is to provide additional context around a patch that serves as a part of a code's history. Don't think of commits as only a way to get code in, but consider them as a way to understand how or why a system came to be as it is. Code is only written once, but needs to be maintained for years to come. As issues arise in the future, in many cases, they may be associated with commit messages from a code base's history. Using commit message context to understand as much as possible about the motivations for changes is an extremely useful debugging tool.

#### Testing a patch

A developer should test their patch before submitted it for review. Patches should be labeled as a work-in-progress by adding (WIP) to the beginning of the initial commit message to indicate that they are not ready for review. This will prevent code reviewers from spending time looking into patches that are not yet ready.

##### Unit tests:

Unit tests are currently written as rspec-puppet. It will eventually be recommended that unit tests be included for all patches, but we are currently not enforcing the requirement.

All unit tests, as well as code validation and lint tests are run every time that a patch is submitted or updated to github. This process uses a backend system called Travisci.

When a patch is submitted or updated, a yellow ball will appear in the github pull request with a link to the associated travis job. Patches will not be merged or reviewed if the travis tests are failing (as indicated by a red ball)

##### Integration tests:

As a convenience method, anyone can use our existing build system to create an environment with their specific patches deployed as a part of a full openstack cluster (note, for advanced users, it is also possible to user vagrant). The current process for building out an environment is to type "please test this" into a github comment for the patch. It is worth noting this is not the intended process, but more-so just a convenience method to perform builds until we finalize a more robust process.

When you type "please test this", it kicks off an associated build in jenkins. A yellow light appears with the word "default in github", you can click on the associated link and it leads you to jenkins. This build builds out a entire openstack environment as VMs in our public cloud with the users patch used as a part of that build process.

NOTE: one of the admins must verify that a user is allowed to submit builds with their patch.

##### Validating that a patch is functional:

It is the developer's responsibility to ensure that their patch is functional. When thinking about functionality, there are a few things to consider:

* when a build is installed from scratch, does my patch result in a fully installed, fully functional system that is configured as expected??
* if the system already exists, does my patch successfully update it to the desired state?

Our current integration tests validate if your patch completely destroys a cluster (ie: affects current behavior), it does not actually verify if your patch has the desired behavior. This is currently done by hand by code reviewers.

NOTE: It is extremely important to ensure that you test your code as much as possible ahead of time. So much of the code review process is about trust, and trust is attained through consistency, thoroughness, and correctness. Consistently writing well thought out patches that are fully tested and include robust commit messages with documentation as required will result in a shorter review time, and more confidence from reviewers if you want to submit more complicated patches in the future. Poorly written, tested, or consistently non-functional patches will either mean that people will not want to review your code.

EVENTUALLY: We will have a much more robust process in place for ensuring that integration tests can be submitted together with a patch so that it is self-validating. This is the desired goal, but a few pieces are still missing (we still don't have an upgrade environment, we don't have a way to validate changes other than vagrant driven validation tests or service checks, code reviewers will currently work with you if your use case requires validation code).

#### Reviewing Code:

Currently, anyone can get involved in the review process through github. Everyone should be able to see (almost - there are a few exception related to production) every code change. I would strongly encourage folks to start watching what patches are going by and comment on anything you have an opinion about.

##### Merge rights:

Although anyone can provide code review, only a few individuals are allowed to merged code. This is a privilege to be earned and not a right of any developer or manager. Remember that any change can potentially result in a production outage, for this reason (and until we have better test coverage), reviewing and merging code can be an extremely complex process for understanding how any given change might related to every single part of the system.

There really isn't any current process for determining who can merge code. There are two main rules:
* It is *never* okay to merge your own code, if you merge your own code, it will be reverted, and you should be reprimanded.
TODO: can we build a process for ensuring that people are reprimanded for merging their own code, eventually, it would be great to reprimand people whose code causes production issues.
* If you don't know if you have permission to merge code, then you don't. Right now, I only feel confident that Harish, Soren, and Dan have the experience to verify and merge things into puppet-rjil (and Harish and Soren both are better equipped to merge things b/c they understand openstack better than Dan does). mayank can merge simple things. Anshup can merge simple things until he gets a little more experience.

##### How to review code

A code reviewer should do the following:

* look at the code in github
* make sure the travis tests are passing
* review the code and commit messages
  * does the commit message provide enough context to understand the changes?
  * does the code appear to be correctly/effectively implemented?
  * does it contain all necessary documentation?
  * does it contain tests?
  * has it considered the upgrade case?
  * does the code effect backwards compatibility? Does it require changes across multiple repositories at the same time?
  * Does the code restrict itself to the scope as defined in the patch?
  * Does the code represent a single logically change?
* Once a reviewer is confident that the code looks correct, he should launch a build by typing "please test this" in the commit field of the patch.
  * verify that the build passes (ie: the code does not break the current system and that it can be installed from scratch)
  * log into the build, and as necessary, verify that the system is in the desired state (ideally, all patches should have self-validating tests, but manually validation of those tests is still a good idea)
  * if necessary, try to test the upgrade case to ensure that it works (NOTE: this is definitely required until we get automated upgrade test running as a part of the pipeline)
  * If something is not working, the reviewer can decide to leave the build alive for the developer to access. In this case, he should update the build reservation form to add that host.
  * If the code reviewer is an approved merger, and he is happy with the code, he can press the merge button and delete the build.
  * Otherwise, he should communicate with the developer of the patch or other team members are required inline in github comments until consensus can be achieved.
  NOTE: ensure that *all* communication related to code patches are made in github so that the history of conversations about code is saved together with the code itself.

#### Build Process

Once code is merged, it enters the build pipeline and slowly works its way to production. Code moves between each stage as the builds from that stage pass. For example, code becomes stuck in a state if the build is not successful and will not proceed to the next step.

##### Acceptance tests:

The first step in the build pipeline is to run acceptance tests. Acceptance tests perform almost exactly the same action as the gate tests (run by please test this). The primary difference is that they test only merged code, specifically, puppet-rjil (and all other components) is installed as a package, and not from git.

Acceptance tests install a fully virtualized environment, then proceed to running validation tests, and finally tempest tests (NOTE: tempest tests are not currently running)

Acceptance tests run every 30 minutes as the [following build](http://jiocloud.rustedhalo.com:8080/job/pipeline-at-deploy/). Only a passing test in acceptance (ie: a green ball) will cause tests to run in the next step. As a side, note, a sequence of failed builds can indicate instability in production, since these builds are run on top of the production environment.

##### Upgrade tests:

Upgrade tests run after acceptance tests have successfully completed. They install the previous version of the build, then apply the changes from the latest version to validate that the upgrade case results in a functional cluster. These tests also run in a virtualized environment.

NOTE: Upgrade tests do not exist yet.

##### Staging tests:

Staging tests perform an upgrade on top of bare-metal in an environment intended to be as identical to production as possible.

NOTE: staging tests do not exist yet.

##### Production update:

Once all builds in the pipeline pass, the production environment is finally updated.

### Upstream projects

NOTE: this document is written by Dan Bode, who has not been involved in the implementation of the package build systems. It contains assumptions about how things *should* look, and is merely meant to serve as a starting place.

At times, developers will need to make changes to code repositories and have these changed deployed to the build system (eventually making it to production).

NOTE: this topic is much more complicated than the topic above, and assumed that the reader has a much better understand of how many things (including git) work.

![Dev process](/images/source-update-build-process.png)

Almost the exact same image is used as was used in the puppet-rjil section, because the overall process is actually the same. The details of each step, however, are very different.

##### writing a patch

In order to implement a source code change, you first need to submit a pull request to the version of the repository forked into the Jiocloud version of the source code. Once this pull request is merged, it will be built into the next version of the packages which will work it's way through the build pipeline.

For example, a patch to nova would need to be submitted to the following [repo](https://github.com/jiocloud/nova).

Patches should also be submitted to the associated upstream repos. It's not clear what the exact policy will need to be, but at a minimum, patches should not be accepted in our forked repository until an associated patch has been submitted upstream. The reason is that patches on the forked version have to be continuously rebased against upstream changes during upgrade. The longer that we hold a patch without it being accepted upstream, the more work this process becomes, and the less we can rely on the upstream testing processes to build confidence about being able to deploy their changes.

##### reviewing a patch

The code review process for a patch is completely undefined. Right now, it's up to Soren (mainly b/c he is the only person that has both a full understanding of all of openstack as well as the build system which is a requirement for understanding change impact), but the reality is that the process needs to be completely different.

##### building packages

Patches get converted into packages to be tested via repoconf(https://github.com/jiocloud/repoconf). Repoconf maintains a separate definition file for every package that we build from our local source repos. It's possible to use this file and the following [job](http://jiocloud.rustedhalo.com:8080/job/pkg_test_build/) to validate that any changes to source code result in a functional build environment. New packages need to be added to repoconf if they are not already being added to package definitions.

NOTE: the above statement is not true, but it needs to be. There needs to be a single repository responsible for defining how packages get converted into packages, but the reality is that it's an entirely undocumented mess of processes that no one except Soren fully understands. In my opinion, the entire package build system just isn't usable until the above is true.

###### Testing packages

The following [job](http://jiocloud.rustedhalo.com:8080/job/pkg_test_build) exists that can be used to test packages from branches of repoconf. All packages that we build need to be moved to repoconf so that we can use this process to test everything.

##### environment builds

The environment build process for how code gets to production is exactly the same as above (puppet-rjil). Basically package repos are created, and used to validate each environment.
