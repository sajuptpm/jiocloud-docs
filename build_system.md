# Introduction

This document is intended to capture the architecture, implementation, and
usage of JioCloud's automated integration, testing, and deployment frameworks.

## What is CI/CD?

CI/CD ( [continuous integration](http://en.wikipedia.org/wiki/Continuous_integration) and [continuos delivery](http://en.wikipedia.org/wiki/Continuous_delivery) ) are
practices through which changes to code repositories are automatically integrated,
verified, and deployed (also called continuos deployment).

# Architecture Overview

The CI/CD system consumes upstream patches and validates them through
a series of steps in a build pipeline. Each step in the pipeline is intended
to perform a set of testing tasks.

## Guiding Design principals

It's easier to understand many of the design decisions made for this project
within the context of our guiding design principals.

1. Automate everything - Humans make too many mistakes to have them perform
   any manual repetitve tasks. Automating tasks also results in fewer mistakes
   and allows tasks to scale.

2. No one should ever make any adhoc modification to our cloud systems. This
   leads to risk that certain systems can wind up in an inconsistent state.
   Machines with inconsistent states are harder to debug at scale and invalidate
   our upgrade testing.

3. Stay as close to master as possible - The closer you are to master, the
   less code you need to deploy to stay up-to-date. This should limit the
   likelyhood that any changeset will lead to a failure.

4. Tests are the gateway for code getting deployed. We have to fully trust
   our automated tests in order to validate that our services are running
   correctly.

5. Build everything expecting it to have to scale massively. Design decisions
   are made assuming that we eventually have to reach 10s or 100s of thousands
   of managed systems. Because of this, we intend to limit the number of
   centralized services as much as possible and expect each of the individual
   servers of our cloud to take on as much of the processing related to their
   own management as possible.

6. Make the deployments as similar as possible for each environment.

## End to End process:

1. Package build server contains a configuration file that tells it
   what external package and code repositories it should be monitoring
   for changes. When changes are detected, it build out new version of
   the updated packages.

2. Package versioning system periodically (every 15 minutes by default)
   monitors the package build server for updates. When updates are detected,
   it builds a new version (snapshot) that represents the latest version of
   all packages (NOTE: snapshots are time based (not patch based) and may
   contain multiple unrelated changes across multiple projects.

3. Jenkins monitors the package versioning system for updates. When updates are
   detected, it initializes a new jenkins job that pushes the latest package
   snapshot version through a build pipeline..

4. The initial job runs what we are calling acceptance tests. These tests
   create a set of virtual machines from scratch and configure them to install
   the desired set of packages. Once the build is verified, tests are run to
   ensure that it results in a functional environment.

5. If this build is successful, it causes more builds to be launched that also
   ensure that servers can be created using the specified snapshot version for:
     * non-functional (performance regression testing)
     * upgrade (verifies that we can upgrade from the
     * staging (verifies that we can provision/upgrade a staging environment
       on bare-metal.
     * production - performs the actual production install/update

## Deployment process

The above section focused on the high level overview of the build pipeline, but ommitted the
details about what the actual environment deployment looks like.


### Deployment inputs

1. The revision of the package repositories to use

All of the configuration detail as well as all packages that get the actual bits on
disk are all stored in the package repository.

2. The specification of what the end enviroment should look like

A single file is used to express how many of each role should exist in the deployment.

which are used to deploy and verify a set of systems.


### Deployment Process

1. The resource file is processed by jiocloud.apply\_resources and results in
all machine required machine being provisioned.

2. Each machine that does not already exist will be provisioned along with a
specified userdata script responsible for:

- installing Puppet
- setting up repositories to the correct version
- running a special Puppet class that is reponsible for install a pre-defined cron job.

3. The desired version for all machines is published into etcd

3. Each machine now has a running cron job that is responsible for managing the state of that machine.
   To do this, the cron job does the following:
* verifies that it can contact etc and derives is its desired snapshot version is greater than it's current
  version
* If it is greater, it does the following:
    * update the repo sources to use the correct version
    * run apt-get update (to update package sources
    * run apt-get upgrade
    * run puppet (to update other non-package update related config)
    * update version in etcd along with pass/fail information

# ToolChain

## Packages

Package are a central component to our design of CI/CD. All code changes
that will be applied to the pipeline will be stored and configured as packages.

Packages allow us to keep track of all changes, and which specific components
they effect. They also allow us to update only the specific bits of a system
that we care about.

Packages allow us to take advantage of upstream security fixes maintained
by Canonical.

## Puppet

Puppet is responsible for configuring of each individual machine into
it's desired roles.

For example, if a machine needed to be configured as a database, Puppet is
responsible for the logic involved in converting a blank machine into this role.

## Openstack API

The openstack API will be used for provisioing all virtual machines and
bare-metal servers required for both test environment as well as production.

### Jenkins

Jenkins is responsible for launching all jobs used as a part of the build
pipeline.

## JioCloud python tools

### python.jiocloud.apply\_resources

The (python-jiocloud)[https://github.com/jiocloud/python-jiocloud]
project contains python helpers tools that are used as a part
of the build process.

#### Apply Resources

The apply resources code is used to manage stacks of VMs
that need to exist in order for a deployment to match the
desired provisioning specification.

#### Apply Command

The apply command is used to evaluate a specification for which
servers/vms should be provisioned in order to fullfill a specification.

The command is as follows:

````
python -m jiocloud.python apply <resource\_file> <userdata> [--project\tag tag]
````

arguments:

* resource\_file

A file that list the specifications of the stack to be created.

#### Delete Command

The delete command is used to delete specific VMs/servers related to a specific
stack.

###

## Installation and Updates

The core infrastructure for running CI/CD is installed and
updated using Puppet and shares as much code and process as possible
with the existing openstack-infra project.

## Developing on CI/CD framework

It is recommended that you test all changes for the CI/CD framework
locally to avoid the likelyhood of pushing code that may disrupt the
function of the CI/CD system.

## CI/CD components


## Build workflow
