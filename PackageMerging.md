Merging
=======

Merging is the process of taking all Ubuntu changes made on top of one Debian version of a package, and re-doing them on top of a new Debian version of the package.

There is a more detailed workflow [here](https://wiki.ubuntu.com/UbuntuDevelopment/Merging/GitWorkflow). This guide is intended to cover the majority of use cases.

You can get a list of packages that have been changed in Debian, but not merged into Ubuntu, here: http://reqorts.qa.ubuntu.com/reports/ubuntu-server/merges.html



Overview
--------

We do merges using `git ubuntu`. As such, the process follows in many ways that of a git rebase, where commits from one point are replayed on top of another point:

    --- something 1.2 ----------------------------- something 1.3
         \                                           \
          -- Ubuntu changes a, b, c -- 1.2ubuntu1     -- Ubuntu changes a, b, c -- 1.3ubuntu1

At a more detailed level, there are other subtasks to be done, such as:

 * Splitting out large "omnibus" style commits into smaller logical units, one commit per logical unit.
 * Harmonizing `debian/changelog` commits into two commits: a changelog merge and a reconstruction.

With this process, we keep the Ubuntu version of a package cleanly applied to the end of the latest Debian version, and make it easy to drop changes as they become redundant.



Process Steps
-------------

 * [Preliminary Steps](#preliminary-steps)
   - [Decide on a merge candidate](#decide-on-a-merge-candidate)
   - [Check existing bug entries](#check-existing-bug-entries)
   - [Make a bug report for the merge](#make-a-bug-report-for-the-merge)
   - [Clone the package repository](#clone-the-package-repository)
 * [The Merge Process](#the-merge-process)
   - [Start a Git Ubuntu Merge](#start-a-git-ubuntu-merge)
   - [Split Commits](#split-commits)
   - [Split out Logical Commits](#split-out-logical-commits)
   - [Prepare the Logical View](#prepare-the-logical-view)
   - [Rebase onto New Debian](#rebase-onto-new-debian)
   - [Finish the Merge](#finish-the-merge)
   - [Fix the Changelog](#fix-the-changelog)
 * [Upload a PPA](#upload-a-ppa)
   - [Get orig tarball](#get-orig-tarball)
   - [Check the source for errors](#check-the-source-for-errors)
   - [Build source package](#build-source-package)
   - [Push to your launchpad repository](#push-to-your-launchpad-repository)
   - [Create a PPA](#create-a-ppa)
 * [Test the New Build](#test-the-new-build)
   - [Package Tests](#package-tests)
   - [Test upgrading from the previous version](#test-upgrading-from-the-previous-version)
   - [Test installing the latest from scratch](#test-installing-the-latest-from-scratch)
   - [Other smoke tests](#other-smoke-tests)
 * [Submit Merge Proposal](#submit-merge-proposal)
 * [Follow Migration](#follow-migration)



Preliminary Steps
-----------------

### Decide on a Merge Candidate

First, you'll need to check that a newer version is even available from Debian. For this, we can use the `rmadison` tool:

    rmadison [package]
    rmadison -u debian [package]

Example:

    $ rmadison at
     at | 3.1.13-1ubuntu1   | precise | source, amd64, armel, armhf, i386, powerpc
     at | 3.1.14-1ubuntu1   | trusty  | source, amd64, arm64, armhf, i386, powerpc, ppc64el
     at | 3.1.18-2ubuntu1   | xenial  | source, amd64, arm64, armhf, i386, powerpc, ppc64el, s390x
     at | 3.1.20-3.1ubuntu2 | bionic  | source, amd64, arm64, armhf, i386, ppc64el, s390x
     at | 3.1.20-3.1ubuntu2 | cosmic  | source, amd64, arm64, armhf, i386, ppc64el, s390x
     at | 3.1.20-3.1ubuntu2 | disco   | source, amd64, arm64, armhf, i386, ppc64el, s390x
    $ rmadison -u debian at
    at         | 3.1.13-2+deb7u1 | oldoldstable       | source, amd64, armel, armhf, i386, ia64, kfreebsd-amd64, kfreebsd-i386, mips, mipsel, powerpc, s390, s390x, sparc
    at         | 3.1.16-1        | oldstable          | source, amd64, arm64, armel, armhf, i386, mips, mipsel, powerpc, ppc64el, s390x
    at         | 3.1.16-1        | oldstable-kfreebsd | source, kfreebsd-amd64, kfreebsd-i386
    at         | 3.1.20-3        | stable             | source, amd64, arm64, armel, armhf, i386, mips, mips64el, mipsel, ppc64el, s390x
    at         | 3.1.23-1        | testing            | source, amd64, arm64, armel, armhf, i386, mips, mips64el, mipsel, ppc64el, s390x
    at         | 3.1.23-1        | unstable           | source, amd64, arm64, armel, armhf, hurd-i386, i386, kfreebsd-amd64, kfreebsd-i386, mips, mips64el, mipsel, ppc64el, s390x
    at         | 3.1.23-1        | unstable-debug     | source

You'll be merging from debian unstable, which in this example is `3.1.23-1`.


### Check existing bug entries

Check for any low hanging fruit in the debian or ubuntu bug list that can be wrapped into this merge.

https://bugs.launchpad.net/ubuntu/+source/[package]

https://tracker.debian.org/pkg/[package]

If there are bugs you'd like to fix, make a new SRU style commit at the end of the merge process and put them together in the same merge proposal.


### Make a bug report for the merge

Search for an existing merge request bug entry in launchpad, and if you don't find one, go to the package's launchpad page (https://bugs.launchpad.net/ubuntu/+source/some-package) and create a new bug report, requesting a merge.

Example:

    URL: https://bugs.launchpad.net/ubuntu/+source/at/+filebug
    Summary: "Please merge 3.1.23-1 into disco"
    Description: "tracking bug"

    result: https://bugs.launchpad.net/ubuntu/+source/at/+bug/1802914

**Save the bug report number, because you'll be using it throughout the merge process.**


### Clone the package repository

    git ubuntu clone <package> [<package>-gu]

Example:

    $ git ubuntu clone at at-gu

It's a good idea to append some git ubuntu specific label (like -gu) to distinguish it from clones of Debian or upstream git repositories (which tend to want to clone as the same name).


The Merge Process
-----------------

### Start a Git Ubuntu Merge

From within the git source tree:

    git ubuntu merge start ubuntu/devel --bug [bug number]

For example:

    $ git ubuntu merge start ubuntu/devel --bug 1802914

This will generate the following tags for you:

| Tag        | Source                                                               |
| ---------- | -------------------------------------------------------------------- |
| old/ubuntu | ubuntu/devel                                                         |
| old/debian | last import tag prior to old/ubuntu without ubuntu suffix in version |
| new/debian | debian/sid                                                           |

The tags themselves will be name-spaced to the current bug in the format `lp12345678`. Thus, for example, your tags may look like:

 * `lp1802914/old/ubuntu`
 * `lp1802914/old/debian`
 * `lp1802914/new/debian`

If `git ubuntu merge start` fails, [do it manually](#start-a-merge-manually)

#### Make a merge branch

Use the merge tracking bug and the ubuntu version it's going into (for example `disco`).

    $ git checkout -b merge-lp1802914-disco
    
If there's no merge bug than the Debian package version you're merging onto can be used (for example `merge-3.1.23-1-disco`)

### Split Commits

In this phase, you split out old-style commits that lumped multiple changes together.

#### Check if there are commits to split

    $ git log --oneline

    2af0cb7 (HEAD -> merge-3.1.20-6-disco, tag: lp1802914/reconstruct/3.1.20-3.1ubuntu2, tag: lp1802914/split/3.1.20-3.1ubuntu2) import patches-unapplied version 3.1.20-3.1ubuntu2 to ubuntu/disco-proposed
    2a71755 (tag: pkg/import/3.1.20-5) Import patches-unapplied version 3.1.20-5 to debian/sid
    9c3cf29 (tag: pkg/import/3.1.20-3.1) Import patches-unapplied version 3.1.20-3.1 to debian/sid
    ...

Get all commit hashes since old/debian, and:

    git show [hash] | diffstat

Example:

    $ git show 2af0cb7 | diffstat
    changelog |    6 ++++++
    control   |    4 ++--
    2 files changed, 8 insertions(+), 2 deletions(-)

Here's another typical example, for the nspr package:

    $ git show d7ebe661 | diffstat
    changelog                                   |  501 ++++++++++++++++++++++++++++
    control                                     |    3 
    patches/fix_test_errcodes_for_runpath.patch |   11 
    patches/series                              |    1 
    rules                                       |    5 
    5 files changed, 520 insertions(+), 1 deletion(-)

Any time you see `changelog` and any other file(s) changing in a single commit, it's guaranteed that you'll need to split it; `changelog` should only ever change in it own commit. You should still look over commits to make sure, but this is a dead giveaway.

Another giveaway would be a commit named `Import patches-unapplied version 1.2.3ubuntu4 to ubuntu/cosmic-proposed`, where it's applying from an ubuntu source rather than a debian one (in this case `ubuntu4`).

If there are no commits to split, simply [add the split tag](#tag-split) and [move on](#prepare-the-logical-view).

#### Identify logical changes

The next step is to separate the changes into logical units.  For the *at* package, this is trivial: just put the changelog change in one commit, and the control change in the other.

The second example, for nspr, is more instructive.  Here we have 5 files changed, that need split out:

 * All changelog changes go to one commit called `changelog`.
 * Update maintainer (in debian/control) goes to one commit called `update maintainers`.
 * All other logically separable commits go into individual commits.

Look in `debian/changelog`:

    nspr (2:4.18-1ubuntu1) bionic; urgency=medium

      * Resynchronize with Debian, remaining changes
        - rules: Enable Thumb2 build on armel, armhf.
        - d/p/fix_test_errcodes_for_runpath.patch: Fix testcases to handle
          zesty linker default changing to --enable-new-dtags for -rpath.

There are two logical changes, which we'll need to separate. Look at the changes in individual files to see which file changes should be logically grouped together.

Example:

    $ git show d7ebe661 -- debian/rules

In this case, we have the following file changes to separate into logical units:

| File(s)            | Logical Unit                                 |
| ------------------ | -------------------------------------------- |
| `debian/rules`     | Enable Thumb2 build on armel, armhf.         |
| `debian/patches/*` | Fix testcases to handle zesty linker default changing to --enable-new-dtags for -rpath.   |
| `debian/control`   | Change maintainer                            |
| `debian/changelog` | Changelog                                    |

#### Split out Logical Commits

Start a rebase at old/debian, and then reset to HEAD^ to bring back the changes as uncommitted changes.

 1. Start a rebase: `git rebase -i lp1803562/old/debian`
 2. Change the commit(s) you're going to split from `pick` to `edit`.
 3. git reset to get your changes back: `git reset HEAD^`

Next, add the commits:

------------------------------------------------------------------------------

Logical Unit:

    $ git add debian/patches/*
    $ git commit

Commit Message:

      * d/p/fix_test_errcodes_for_runpath.patch: Fix testcases to handle
        zesty linker default changing to --enable-new-dtags for -rpath.

------------------------------------------------------------------------------

Logical Unit:

    $ git add debian/rules
    $ git commit

Commit Message:

      * d/rules: Enable Thumb2 build on armel, armhf.

------------------------------------------------------------------------------

Maintainers:

    $ git commit -m "update maintainers" debian/control

------------------------------------------------------------------------------

Changelog:

    $ git commit -m changelog debian/changelog

------------------------------------------------------------------------------

Finally, complete the rebase:

    $ git rebase --continue


#### Tag Split

Note: Do this even if there were no commits to split.

    $ git ubuntu tag --split --bug 1803562


### Prepare the Logical View

In this phase, we make a clean, "logical" view of the history. This history is cleaned up (but has the same delta), and only contains the actual changes that affect the package's behavior.

Start a rebase from old/debian:

    $ git rebase -i lp1803562/old/debian

Now we do some cleaning:

* Delete imports, etc
* Delete changelog, maintainer
* Possibly rearrange commits if it makes logical sense

You should also squash these kinds of commits together:

 * Changes and reversions of those changes, since they resolve to a no-op.
 * Multiple changes to the same patch file, since they should be a logical unit.

To squash a commit, move its line underneath the one you want it to become part of, and then change it from `pick` to `fixup`.

#### Check the result

At the end of the squash and clean phase, the only delta you should see from the split tag is:

    $ git diff lp1803562/split/2%4.18-1ubuntu1 |diffstat
     changelog |  762 --------------------------------------------------------------
     control   |    3 
     2 files changed, 1 insertion(+), 764 deletions(-)

Only changelog and control were changed, which is what we want.

#### Create logical tag

    $ git ubuntu tag --logical --bug 1803562

This may fail with an error like: `ERROR:HEAD is not a defined object in this git repository.`, in which case [do it manually](#create-logical-tag-manually)


### Rebase onto New Debian

    $ git rebase -i --onto lp1803562/new/debian lp1803562/old/debian

#### Conflicts

If a conflict occurs, you must resolve it. We do so by modifying the conflicting commit during the rebase.

An example, merging logwatch 7.5.0-1:

    $ git rebase -i --onto lp1810928/new/debian lp1810928/old/debian
    ...
    CONFLICT (content): Merge conflict in debian/control
    error: could not apply c0efd06... - Drop libsys-cpu-perl and libsys-meminfo-perl from Recommends to
    ...

Take a look at the conflict in debian/control:

    <<<<<<< HEAD
    Recommends: libdate-manip-perl, libsys-cpu-perl, libsys-meminfo-perl
    =======
    Recommends: libdate-manip-perl
    Suggests: fortune-mod, libsys-cpu-perl, libsys-meminfo-perl
    >>>>>>> c0efd06... - Drop libsys-cpu-perl and libsys-meminfo-perl from Recommends to

Upstream removed `fortune-mod`, and deleted the entire line since it was no longer needed. Resolve it to:

    Recommends: libdate-manip-perl
    Suggests: libsys-cpu-perl, libsys-meminfo-perl

Continue with the rebase:

    $ git add debian/control
    $ git rebase --continue

#### Empty commits

If a commit becomes empty, it's because the change has already been applied upstream:

    The previous cherry-pick is now empty, possibly due to conflict resolution.

In such a case, the commit can be dropped.

    $ git rebase --abort
    $ git rebase -i lp1803296/old/debian

Keep a copy of the unneeded commit's commit message, then delete it in the rebase.

#### Sync request

If all the commits are empty, or you realized there aren't logical changes, you're facing a **Sync Request**, not a merge anymore. Check [this](Syncs.md) to continue.

#### Check that the patches still apply cleanly:

    $ quilt push -a --fuzz=0

##### If quilt fails:

Quilt can fail at this point if the file being patched has changed significantly upstream. The most common reason is that the issue the patch addresses has since been fixed upstream.

For example:

    $ quilt push -a --fuzz=0
    ...
    Applying patch ssh-ignore-disconnected.patch
    patching file scripts/services/sshd
    Hunk #1 FAILED at 297.
    1 out of 1 hunk FAILED -- rejects in file scripts/services/sshd
    Patch ssh-ignore-disconnected.patch does not apply (enforce with -f)

If this patch fails because the changes in `ssh-ignore-disconnected.patch` are already applied upstream, you must remove this patch.

    $ git log --oneline

    1aed93f (HEAD -> ubuntu/devel)   * d/p/ssh-ignore-disconnected.patch: [sshd] ignore disconnected from user     USER (LP: 1644057)
    7d9d752 - Drop libsys-cpu-perl and libsys-meminfo-perl from Recommends to   Suggests as they are in universe.

Removing `1aed93f` will remove the patch.

 * Save the commit message from `1aed93f` for later including in the `Dropped Changes` section of the new changelog entry.
 * `git rebase -i 7d9d752` and delete commit `1aed93f`.

#### Unapply patches before continuing

    $ quilt pop -a


### Finish the Merge

    $ git ubuntu merge finish ubuntu/devel --bug 1803296

* If this fails, [do it manually](#finish-the-merge-manually)


### Fix the Changelog

Git ubuntu attempts to put together a changelog entry, but it will likely have problems. Fix it up to make sure it follows the standards. See [Committing your Changes](CommittingChanges.md) for information about what it should look like.

#### Add dropped changes

If you dropped any changes (due to upstream fixes), you must note them in the changelog entry:

      * Dropped Changes:
        - Foo: change to bar
          [Fixed in 1.2.3-4]

#### Commit the changelog fix:

    $ git commit debian/changelog -m changelog

#### Rebase and squash changelog

Now you must rebase and squash the changelog changes into the "reconstruct-changelog" commit. Do a rebase with the new/debian tag:

    $ git rebase -i lp1810928/new/debian

Change the order from:

    pick e431327 merge-changelogs
    pick cf3b93a reconstruct-changelog
    pick 21cea1a update-maintainer
    pick 08c2f4d changelog

to:

    pick e431327 merge-changelogs
    pick cf3b93a reconstruct-changelog
    f 08c2f4d changelog
    pick 21cea1a update-maintainer

Notice above that you must also change the `changelog` rebase command to `fixup` (or `f`).



Upload a PPA
------------

### Get orig tarball

Ubuntu doesn't know about the new tarball yet, so we must create it.

    $ git ubuntu export-orig

* If this fails, [do it manually](#get-orig-tarball-manually)


### Build source package

    $ dpkg-buildpackage -S -nc -d -sa -v3.1.20-3.1ubuntu2

The switches are:

 * -S = build source only
 * -nc = no clean
 * -d = ?
 * -sa = include orig tarball (required on a merge)
 * -vXYZ = include changelog since XYZ

Changes should be from the last ubuntu version

#### Check the built package for errors

    $ lintian --pedantic --display-info --verbose --info --profile ubuntu ../at_3.1.23-1ubuntu1.dsc


### Push to your launchpad repository

Now that the package is tested and builds successfully, it's time to push it to your launchpad repository.

The easiest way is to run it like this:

    git push your-lp-username

You'll get an error message and a suggestion for how to set upstream. For example:

    $ git push kstenerud
    fatal: The current branch merge-lp1802914-disco has no upstream branch.
    To push the current branch and set the remote as upstream, use

        git push --set-upstream kstenerud merge-lp1802914-disco

Run the suggested command to push to your repository.

#### Push your lp tags

    $ git push kstenerud $(git tag |grep 1802914 | xargs)
    To ssh://git.launchpad.net/~kstenerud/ubuntu/+source/at
     * [new tag]         lp1802914/split/3.1.20-3.1ubuntu2 -> lp1802914/split/3.1.20-3.1ubuntu2
     * [new tag]         lp1802914/logical/3.1.20-3.1ubuntu2 -> lp1802914/logical/3.1.20-3.1ubuntu2
     * [new tag]         lp1802914/new/debian -> lp1802914/new/debian
     * [new tag]         lp1802914/old/debian -> lp1802914/old/debian
     * [new tag]         lp1802914/old/ubuntu -> lp1802914/old/ubuntu
     * [new tag]         lp1802914/reconstruct/3.1.20-3.1ubuntu2 -> lp1802914/reconstruct/3.1.20-3.1ubuntu2


### Create a PPA

You'll need to have a PPA for reviewers to test.

#### Create a PPA repository

https://launchpad.net/~kstenerud/+activate-ppa

Give it a name that identifies the ubuntu version, package name, and bug number, such as `at-merge-lp1802914`

**IMPORTANT:** Be sure to enable all architectures to check that it builds (click on `Change details` in the top right corner of the newly created PPA page).

#### Upload files

    $ dput ppa:kstenerud/at-merge-lp1802914 ../at_3.1.23-1ubuntu1_source.changes

#### Wait for packages to be ready

Check the PPA page to see when packages are finished building: https://launchpad.net/~kstenerud/+archive/ubuntu/at-merge-lp1802914

Also, look at the package contents to make sure they have actually been published: https://launchpad.net/~kstenerud/+archive/ubuntu/at-merge-lp1802914/+packages



Test the New Build
------------------

Test the following:

    1. Package Tests
    2. Upgrading from the previous version
    3. Installing the latest where nothing was installed before
    4. Other smoke tests


### Package Tests

[Run package tests (if any)](PackageTests.md)


### Test upgrading from the previous version

Example:

Note: Disco is not yet available at the time of writing, so we use cosmic.

    $ lxc launch images:ubuntu/cosmic tester && lxc exec tester bash
    $ apt update && apt dist-upgrade -y && apt install -y at

The test:

    echo "echo xyz >test.txt" |at now + 1 minute && sleep 1m && cat test.txt && rm test.txt

Upgrade:

    $ add-apt-repository -y ppa:kstenerud/at-merge-lp1802914

Note: Disco is not yet available at the time of writing, so we must modify the source list entry:

    $ vi /etc/apt/sources.list.d/kstenerud-ubuntu-at-merge-lp1802914-cosmic.list
    * change cosmic to disco

    $ apt update && apt dist-upgrade -y

Test the upgraded version:

    $ echo "echo abc >test.txt" | at now + 1 minute && sleep 1m && cat test.txt && rm test.txt


### Test installing the latest from scratch

    $ lxc launch images:ubuntu/cosmic tester && lxc exec tester bash
    $ add-apt-repository -y ppa:kstenerud/at-merge-lp1802914
    $ apt update && apt dist-upgrade -y && apt install at
    $ echo "echo abc >test.txt" | at now + 1 minute && sleep 1m && cat test.txt && rm test.txt


### Other smoke tests

 * Try running various basic commands.
 * Try running regression tests: https://git.launchpad.net/qa-regression-testing



Submit Merge Proposal
---------------------

NOTE: Git branch with % in name doesn't work. Use something like _

    $ git ubuntu submit --reviewer canonical-server-packageset-reviewers --target-branch debian/sid
    Your merge proposal is now available at: https://code.launchpad.net/~kstenerud/ubuntu/+source/at/+git/at/+merge/358655
    If it looks OK, please move it to the 'Needs Review' state.

* If this fails, [do it manually](#submit-merge-proposal-manually)


### Update the merge proposal

 * Link the PPA
 * add any other info that will help reviewer to mp as a comment.

Example:

    PPA: https://launchpad.net/~kstenerud/+archive/ubuntu/at-merge-lp1802914

    Basic test:

    $ echo "echo abc >test.txt" | at now + 1 minute && sleep 1m && cat test.txt && rm test.txt

    Package tests:

    This package contains no tests.


#### Open the review

Change the MP status from "work in progress" to "needs review"



Follow Migration
----------------

Once the merge proposal goes through, you must follow the package to make sure it gets to its destination.

### Package Tests

The results from the latest package tests will be published for each ubuntu release.

For example: http://autopkgtest.ubuntu.com/packages/o/openssh/cosmic/amd64

### Proposed Migration

The status of all packages will be available from http://people.canonical.com/~ubuntu-archive/proposed-migration or one of its subdirectories. The top level directory is for the current dev release. Previous releases are in subdirectories.



-----------------------------------------------------------------------------

Manual Steps
------------

### Start a merge manually

#### Generate the merge branch

Create a branch to do the merge work in:

    $ git checkout -b merge-lp1802914-disco


#### Create tags

| Tag        | Source             |
| ---------- | ------------------ |
| old/ubuntu | ubuntu/disco-devel |
| old/debian | last import tag prior to old/ubuntu without ubuntu suffix in version    |
| new/debian | debian/sid         |

per: https://www.debian.org/releases/
"debian/sid" always matches to debian unstable.

You can find the last import tag using `git log | grep "tag: pkg/import" | grep -v ubuntu | head -1`:

    ...
    commit 9c3cf29c05c3fddd7359e71c978ff9a9a76e4404 (tag: pkg/import/3.1.20-3.1)

So, we create the following tags:

    $ git tag lp1802914/old/ubuntu pkg/ubuntu/disco-devel
    $ git tag lp1802914/old/debian 9c3cf29c05c3fddd7359e71c978ff9a9a76e4404
    $ git tag lp1802914/new/debian pkg/debian/sid


#### Start a rebase

    $ git rebase -i lp1802914/old/debian

#### Clear any history prior to, and including the last debian version

If the package hasn't been updated since the git repository structure has changed, it will grab all changes throughout time rather than since the last debian version. Simply delete the older lines from the interactive rebase

In this case, up to, and including import of 3.1.20-3.1

#### Create reconstruct tag

    $ git ubuntu tag --reconstruct --bug 1802914

Next step: [Split Commits](#split-commits)


### Create logical tag manually

Use the version number of the last ubuntu change. So if there are `3.1.20-3.1ubuntu1` and `3.1.20-3.1ubuntu2`, use `3.1.20-3.1ubuntu2`.

    $ git tag -a -m "Logical delta of 3.1.20-3.1ubuntu2" lp1802914/logical/3.1.20-3.1ubuntu2

Note: Certain characters aren't allowed in git. For example, `:` should be replaced with `%`.

Next step: [Rebase onto new debian](#rebase-onto-new-debian)


### Finish the merge manually

Merge changelogs of old ubuntu and new debian:

    $ git show lp1802914/new/debian:debian/changelog >/tmp/debnew.txt
    $ git show lp1802914/old/ubuntu:debian/changelog >/tmp/ubuntuold.txt
    $ merge-changelog /tmp/debnew.txt /tmp/ubuntuold.txt >debian/changelog 
    $ git commit -m "Merge changelogs" debian/changelog

Create new changelog entry for the merge:

    $ dch -i

Which creates for example:

    at (3.1.23-1ubuntu1) disco; urgency=medium

      * Merge with Debian unstable (LP: #1802914). Remaining changes:
        - Suggest an MTA rather than Recommending one.

     -- Karl Stenerud <karl.stenerud@canonical.com>  Mon, 12 Nov 2018 18:11:53 +0100

Commit the changelog:

    $ git commit -m "changelog: Merge of 3.1.23-1" debian/changelog

Update maintainer:

    $ update-maintainer
    $ git commit -m "Update maintainer" debian/control

Next step: [Fix the Changelog](#fix-the-changelog)


### Get orig tarball manually

    $ git checkout -b pkg/importer/debian/pristine-tar
    $ pristine-tar checkout at_3.1.23.orig.tar.gz
    $ git checkout merge-3.1.23-1-disco

#### If git checkout also fails:

    $ git checkout merge-lp1802914-disco
    $ cd /tmp
    $ pull-debian-source at
    $ mv at_3.1.23.orig.tar.gz ~/work/packages/ubuntu/
    $ cd -

Next step: [Check the source for errors](#check-the-source-for-errors)


### Submit merge proposal manually

    $ git push kstenerud merge-lp1802914-disco

Then, create a MP manually in launchpad, and save the URL.

Next step: [Update the merge proposal](#update-the-merge-proposal)
