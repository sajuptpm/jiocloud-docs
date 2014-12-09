# introduction

This document is inteded to capture the package build process

## Package build process

NOTE: this part of the document is not as up-to-date as other parts
and needs more information about the exact details.

There is one jenkins system (rusted halo) that does the folowing:
  - monitors openstack changes for updates
  - runs unit tests
  - builds source package (runs unit tests again)
  - updates rustedhalo repo

There is an internal jenkins repo that does the following:
  - monitors multiple package sources to mirror (ubuntu, ceph, puppetlabs)
  - runs every 30 minutes and runs refresh mirror
  - builds a new snapshot

### End to End process:

1. Package build server contains a configuration file that tells it
   what external package and code repositories it should be monitoring
   for changes. Jobs exist that pull in the current version of those
   repositories and run their unit tests. Once the unit tests are validated,
   the latest version of the set of external repositories is packaged
   as a single versioned repository.

2. Package versioning system periodically (every 30 minutes by default)
   monitors the package build server for updates. When updates are detected,
   it builds a new version (snapshot) that represents the latest version of
   all packages (NOTE: snapshots are time based (not patch based) and may
   contain multiple unrelated changes across multiple projects.

3. Jenkins monitors the package versioning system for updates. When updates are
   detected, it initializes a new jenkins job that pushes the latest package
   snapshot version through the build pipeline.

4. The initial job runs what we are calling acceptance tests. These tests
   create a set of virtual machines from scratch and configure them to install
   the desired set of packages. Once the build is verified, tests are run to
   ensure that it results in a functional environment.

5. If this build is successful, the next steps of the pipeline are executed on
   the same snapshot version.


## puppet-rjil and python-rjil

## Package promotion process (for puppet-rjil and python-rjil)

NOTE - does this apply to other packages? (like openstack)


When a new package is built (e.g. when we merge a new change into puppet-rjil,
Jenkins sees that and builds a .deb out of it) it lands in trusty-unstable immediately.

At the start of an acceptance test run (not the gate tests!), everything in trusty-unstable
is "promoted" into trusty-testing.

The acceptance test run uses trusty-testing.

gate tests should run using trusty-unstable, but it doesn't quite work yet.

If the acceptance test run is succesful, everything in trusty-testing is
promoted to trusty (the stable repo).

The stable repo is what gets mirrored to the staging/production environments.

This ensures two things:
* Acceptance test runs have a stable base. Nothing new lands in trusty-testing during a test run.
* Things only get promoted to trusty (stable) in combination with packages that they've been tested with.

I.e. we can't accidentally get python-jiocloud into staging before the corresponding puppet-rjil changes have landed.
