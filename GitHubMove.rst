==============================
Moving LLVM Projects to GitHub
==============================

Introduction
============

This is a proposal to move our current revision control system from our own
hosted Subversion to GitHub. Below are the financial and technical arguments as
to why we need such a move and how will people (and validation infrastructure)
continue to work with a Git-based LLVM.

There will be a survey pointing at this document when we'll know the community's
reaction and, if we collectively decide to move, the time-frame. Be sure to
make your view count.

Essentially, the proposal is divided in the following parts:

* Outline of the reasons to move to Git and GitHub
* Description on the options
* What some examples of workflow will look like (compared to currently)
* The proposed migration plan

What This Proposal is *Not* About
=================================

The development of LLVM will continue as it exists now, with the same policy.

This proposal aims at focusing on moving the hosting of the repository from SVN
to git. To this end, this document focuses on this single goal and specifically
excludes any workflow change. I.e. it should not be assumed that moving to
GitHub implies using the GitHub issue tracking, or using the GitHub UI for
pull-requests and/or code-review.

Every existing contributors will get commit access on demand. Those who don't
have an existing GitHub account will have to create one in order to continue
having commit access to any sub-projects.

Why Git, and Why GitHub?
========================

Why Move At All?
----------------

The strongest reason for the move, and why this discussion started in the first
place, is that we currently host our own Subversion server and Git mirror in a
voluntary basis. The LLVM Foundation sponsors the server and provides limited
support, but there is only so much it can do.

Volunteers are not Sysadmins themselves, but compiler engineers that happen
to know a thing or two about hosting servers. We also don't have 24/7 support,
and we sometimes wake up to see that continuous integration is broken because
the SVN server is either down or unresponsive.

On the other hand, there are multiple services out there (GitHub, GitLab,
BitBucket among others) that offer that same service (24/7 stability, disk
space, Git server, code browsing, forking facilities, etc) for the very
affordable price of *free*.

Why Git?
--------

Most new coders nowadays start with Git. A lot of them have never used SVN, CVS,
or anything else. Websites like GitHub have changed the landscape of open source
contributions, reducing the cost of first contribution and fostering
collaboration.

Git is also the version control multiple LLVM developers use. Despite the
sources being stored in a SVN server, some LLVM developers are already using git
through the Git-SVN integration.

Git features allow you to:

* Commit, squash, merge, fork locally without any penalty to the server
* Add as many branches as necessary to allow for multiple threads of development
* Inspect the repository history (blame, log, bisect) without Internet access

In addition, because Git seems to be replacing every project's version control
system and because it's flexibility, there are more tools that are built over
Git. Future tooling is much more likely to support Git first (if not only), than any
other version control system.

Why GitHub?
-----------

GitHub, like GitLab and BitBucket, provide free code hosting for open source
projects. Any of these can replace the infrastructure that we have today that
serves code repository and user control access.

They also have a dedicated team to monitor, migrate, improve and distribute the
contents of the repositories depending on region and load.

GitHub offers read-write **SVN** access to the repository
(https://github.com/blog/626-announcing-svn-support).
This would enable people that still have/want to use SVN infrastructure and
tooling to slowly migrate or even stay working as if it was an SVN repository.

So, any of the three solutions solve the cost and maintenance problem, but
GitHub has an additional feature that would be beneficial to the migration
plan. Moreover, there are already multiple LLVM mirrors on GitHub which seems to
indicate that part of the community already settled there.

On Managing Revision Numbers with Git
=====================================

The current SVN repository hosts all the LLVM sub-projects alongside of each
other. This unified SVN repository provides a single monotonically increasing
integer revision number that is synchronized for every sub-projects: revision
r279422 allows to checkout LLVM and all the sub-projects in sync.

This property has been considered useful in past discussions in the community
about git, for example:

- "The 'branch' I most care about is mainline, and losing the ability to say
  'fixed in r1234' (with some sort of monotonically increasing number) would
  be a tragic loss." [LattnerRevNum]_
- "I like those results sorted by time and the chronology should be obvious, but
  timestamps are incredibly cumbersome and make it difficult to verify that a
  given checkout matches a given set of results." [TrickRevNum]_
- "There is still the major regression with unreadable version numbers.
  Given the amount of Bugzilla traffic with 'Fixed in...', that's a
  non-trivial issue." [JSonnRevNum]_
- "Sequential IDs are important for LNT and llvmlab bisection tool." [MatthewsRevNum]_.

Git does not use sequential integer revision number but instead uses a hash to
identify each commit (Linus mentioned that the lack of such revision number
is "the only real design mistake" in git [TorvaldRevNum]_.

However, git can emulate this increasing revision number:
`git rev-list  --count <commit-hash>`. The main caveat is that this does
work only inside a single branch (like our current trunk) and only inside a
single repository. However, having a revision number that increases across
branches has never been mentioned as something desirable, or enabling anything
interesting.

As a conclusion, we'll make sure `clang -v` reports a `useful` revision number
(e.g. `master-12345` or `4.0-5321`). This solution should be enough to address
the objections raised above on this aspect.

One or Multiple Repositories
============================

Two major proposals are considered:

1. Moving every single SVN sub-projects into its own separate git repository.
2. Moving *all* the LLVM projects into a single repository.

The first proposal would mimic the existing official separate read-only git
repositories (i.e. for example http://llvm.org/git/compiler-rt.git), while the
second one would mimic an export of the SVN repository
(i.e. would look like https://github.com/llvm-project/llvm-project) where every
sub-projects is alongside each other.
In the latter case, the existing read-only repositories (i.e. for example
http://llvm.org/git/compiler-rt.git) with git-svn read-write access would be
maintained

There are other impacts that are less immediates and less technicals: the first
proposal of keeping the repository separate implies a view where the
sub-projects are very independent and isolated, while the second proposal
encourage better code sharing and refactoring across projects, for example
reusing a datastructure initially in LLDB by moving it into libSupport. It
would also be very easy to decide to extract some pieces of libSupport and/or
ADT to a new top-level *independent* library that can be reused in libcxxabi for
instance. Finally, it also encourages to update all the subprojects when
changing API or refactoring code ("git grep" works across sub-projects for
instance).

Some concerns have been raised that having a single repository would be a burden
for downstream users that have interest in only a single repository, however
this is addressed by keeping a read-only git repo for each project just as we
do today. Also the GitHub SVN bridge allows to contribute to a single
sub-project the same way it is possible today.

How Do We Handle A Single Revision Number Across Multiple Repositories
----------------------------------------------------------------------

A key need is to be able to checkout multiple projects (i.e. lldb+llvm or
clang+llvm+libcxx for example) at a given revision.

The second proposal keeps naturally the property of the existing SVN repository:
the sub-projects move synchronously and a single revision number identifies the
state of the development across all projects.

The first proposal, with separate is more involved and requires tooling to
achieve this. We describe here the proposed solution.

Fundamentally, separated git repositories imply that a tuple of revisions
(one entry per repository) is needed to describe the state across
repositories/sub-projects.
For example, a given version of clang would be
*<LLVM-12345, clang-5432, libcxx-123, etc.>*.
To make this more convenient, a separate *umbrella* repository would be
provided. This repository would be used for the sole purpose of understanding
the *sequence* (with some granularity) in which commits were added across
repository and provide a single revision number.

This *umbrella* repository will be read-only and periodically updated
to record the above tuple. The proposed form to record this is to use git
[submodules]_, possibly along with a set of scripts to help check out a
specific revision of the LLVM distribution. A regular LLVM developer does not
need to interact with the *umbrella* repository, the individual repositories
can be checked out independently.

One example of such a repository is Takumi's llvm-project-submodule
(https://github.com/chapuni/llvm-project-submodule), which when checked out,
will have the references to all sub-modules but not check them out, so one will
need to *init* the sub-modules manually. This will allow the same behavior
as checking out individual SVN repositories, as it will keep the
cross-repository history.

This umbrella repository will be updated automatically by a bot (running on
notice from a webhook on every push, and periodically). Note that commits in
different repositories pushed within the same time frame may be visible together
or in undefined order in the umbrella repository.

What About Branches and Merges?
===============================

Contrary to SVN, git makes it easy and encourages branches. The commits history
is represented as a DAG, which is a departure from SVN linear history. As part
of the current plan, we'll *enforce linear history* in the repository (this
applies for both proposal).

The implementation will rely on a combination of client-side and server-side
hooks. GitHub offers a feature called `Status Checks`: a branch protected by
`status checks` requires commits to be *white-listed* before the push can happen.
A supplied pre-push hook on the client side will run and check the history,
before white-listing the commit being pushed [statuschecks]_.

What About Commits Emails / Mailing-list?
=========================================

An extra bot will need to be setup to continue to send emails for every commit.
We'll keep the exact same email format as the existing (a change is possible
later, but beyond the scope of the current discussion), the only difference
being replacing the URL from "http://llvm.org/viewvc/..." to
"http://github.org/llvm...".

Workflow Before/After
=====================

This section goes through a few examples of workflows.

Checkout/Clone a Single Project, without Commit Access
------------------------------------------------------

Except the URL, nothing changes. The possibilities today are::

  svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
  # or with git
  git clone http://llvm.org/git/llvm.git

With GitHub you would do either::

  git clone https://github.com/llvm-project/llvm.git
  # or using the GitHub svn native bridge
  svn co https://github.com/llvm-project/llvm/trunk

This is valid for both proposal, as we'll maintain a read-only view of the
individual subprojects repos.

Checkout/Clone a Single Project, with Commit Access
---------------------------------------------------

Currently
^^^^^^^^^
::

  # direct SVN checkout
  svn co https://user@llvm.org/svn/llvm-project/llvm/trunk llvm
  # or using the read-only git view, with git-svn
  git clone http://llvm.org/git/llvm.git
  cd llvm
  git svn init https://llvm.org/svn/llvm-project/llvm/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l  # -l avoids fetching ahead of the git mirror.

Commits are performed using "svn commit" or "git commit" and "git svn dcommit".

Split Repositories (Submodules) Proposal
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

With the first proposal, nothing changes but the URL, and commits can be
performed using "svn commit" or "git commit" and "git push"::

  git clone https://github.com/llvm/llvm.git llvm
  # or using the GitHub svn native bridge
  svn co https://github.com/llvm/llvm/trunk/ llvm

Single Repository Proposal
^^^^^^^^^^^^^^^^^^^^^^^^^^

With the second proposal, there are multiple possibilities to achieve this.
First it is possible to clone the full repository::

  git clone https://github.com/llvm/llvm-projects.git llvm
  # or using the GitHub svn native bridge
  svn co https://github.com/llvm/llvm-projects/trunk/ llvm

At this point you have every sub-projects (llvm, clang, lld, lldb, ...), which
**doesn't imply you have to build all of them**. You can still build **only**
compiler-rt for instance. It is **not** different from someone who would
checkout all the projects with SVN today. You can commit regularly in a single
subproject using "git commit" and "git push" or "svn commit", and read the
history for a single project (*git log libcxx* for example).

If you really don't want to have the sources for all the sub-projects checked
out for any reason, there are again a few options.
First using git sparse checkout::

  mkdir llvm
  cd llvm
  git init
  git remote add origin https://github.com/joker-eph/llvm-unified/
  git config core.sparseCheckout true
  mkdir .git/info
  echo /compiler-rt >> .git/info/sparse-checkout
  git pull origin master

This actually fetch the data, and checkout **only** the compiler-rt sources.

Secondly, using GitHub svn native bridge::

  svn co https://github.com/llvm/llvm-projects/trunk/compiler-rt compiler-rt  —username=...

This checks out only compiler-rt and provides commit access using "svn commit",
in the **exact** same way as it would do today.

Finally using *git-svn* from one of the read-only git repo::

  # Clone from the single read-only git repo
  git clone http://llvm.org/git/llvm.git
  cd llvm
  # Configure the SVN remote and initialize the svn metadata
  $ git svn init https://github.com/joker-eph/llvm-project/trunk/llvm —username=...
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l

In this case the repository contains only a single subproject and commits can be
made using "git svn dcommit", again **just as we do today**.

Checkout/Clone Multiple Projects, with Commit Access
----------------------------------------------------

Let's look how to assemble llvm+clang+libcxx at a given revision.

Currently
^^^^^^^^^
::

  svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm -r $REVISION
  cd llvm/tools
  svn co http://llvm.org/svn/llvm-project/clang/trunk clang -r $REVISION
  cd ../projects
  svn co http://llvm.org/svn/llvm-project/libcxx/trunk libcxx -r $REVISION

Or using git-svn::

  git clone http://llvm.org/git/llvm.git
  cd llvm/
  git svn init https://llvm.org/svn/llvm-project/llvm/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l
  git checkout `git svn find-rev -B r258109`
  cd tools
  git clone http://llvm.org/git/clang.git
  cd clang/
  git svn init https://llvm.org/svn/llvm-project/clang/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l
  git checkout `git svn find-rev -B r258109`
  cd ../../projects/
  git clone http://llvm.org/git/libcxx.git
  cd libcxx
  git svn init https://llvm.org/svn/llvm-project/libcxx/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l
  git checkout `git svn find-rev -B r258109`

Note that the list would be longer with more subprojects.

Split Repositories (Submodules) Proposal
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

With the submodule proposal, the umbrella repository enters the dance. This is
where the mapping from a single revision number to the individual repositories
revisions is stored.::

  git clone https://github.com/llvm-beanz/llvm-submodules
  cd llvm-submodules
  git checkout $REVISION
  git submodule init
  git submodule update clang llvm libcxx

At this point the clang, llvm, and libcxx individual repositories are cloned and
stored alongside each other. Some CMake options can cope with this, otherwise
creating symlinks to fit the magic discovery of projects by CMake can work as
well.

Single Repository Proposal
^^^^^^^^^^^^^^^^^^^^^^^^^^

The repository contains natively the source for every sub-projects at the right
revision, which makes this straightforward::

  git clone https://github.com/llvm/llvm-projects.git llvm
  cd llvm
  git checkout $REVISION

As previous, at this point clang, llvm, and libcxx are stored in directories
alongside each other. Some CMake options can deal with this, otherwise
creating symlinks to fit the magic discovery of projects by CMake can work as
well.

Commit an API Change in LLVM and Update the Sub-projects
--------------------------------------------------------

While it is technically possible today, it is complicated enough that most
people are not trying to update LLD or Clang in the same commit as the API is
changed in LLVM for example.

The split repositories (submodules) proposal does not address this: one would
have to commit and push separately in every individual repository. The umbrella
repository may or may not group these individual commits in the same revision.

The single repository proposal handles this natively and makes this use case
trivial.

Branching/Stashing/Updating for Local Development or Experiments
----------------------------------------------------------------

Currently
^^^^^^^^^

SVN does not allow this use case, but developers that are currently using
git-svn can do it. Let's look in practice what it means when dealing with
multiple sub-projects. First the same initial checkout as before::

  git clone http://llvm.org/git/llvm.git
  cd llvm/
  git svn init https://llvm.org/svn/llvm-project/llvm/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l
  cd tools
  git clone http://llvm.org/git/clang.git
  cd clang/
  git svn init https://llvm.org/svn/llvm-project/clang/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l
  cd ../../projects/
  git clone http://llvm.org/git/libcxx.git
  cd libcxx
  git svn init https://llvm.org/svn/llvm-project/libcxx/trunk --username=<username>
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l

To refresh the repository with ToT::

  git pull
  cd tools/clang
  git pull
  cd ../../projects/libcxx
  git pull

To create a new branch::

  git checkout -b MyBranch
  cd tools/clang
  git checkout -b MyBranch
  cd ../../projects/libcxx
  git checkout -b MyBranch

To switch branches::

  git checkout AnotherBranch
  cd tools/clang
  git checkout -b AnotherBranch
  cd ../../projects/libcxx
  git checkout -b AnotherBranch

Split Repositories (Submodules) Proposal
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The split repository is on the same level as the existing read-only git
views of the SVN repository: every commands needs to be applied to the
individual repositories.

Single Repository Proposal
^^^^^^^^^^^^^^^^^^^^^^^^^^

All these manipulations are naturally handled by the straightforward version
of the git sub-commands:

To refresh the repository with Tot::

  git pull

To create a new branch::

  git checkout -b MyBranch

To switch branches::

  git checkout AnotherBranch

Bisecting
---------

FIXME: TODO.

Living Downstream
-----------------

For integrators and downstream projects, multiple solutions are possible to
continue integrating:

1. Pull from SVN. If you were pulling from the SVN repo yesterday, you can
   continue to use SVN. However, the revision numbers **will** change and this
   may break your integration.
2. Pull from individual git repositories for each projects. If you were pulling
   you integration from one of the existing repo, this should still be possible
   in the future as a read-only view of the individual projects will be
   maintained.
3. Migrate to a unified repository (proposal two). It has been shown as an
   experiment using the "Cherry" project how it can be performed, both by
   rewriting the git history of the project [LebarCherry]_ or preserving
   it [AminiCherry]_.

FIXME: more details? For example how to upstream internal patches?

Variant
=======

A variant is to group together in a single repository only the projects that are
*rev-locked* to LLVM (clang, lld, lldb, ...) and leave projects like libcxx and
compiler-rt in their own individual and separate repository.

It is not clear if we would still really need an umbrella repository in this
configuration.

Previews
========

FIXME: make something more official/testable and update all the URLs in the
examples above.

Example of a working version:

* Repository: https://github.com/llvm-beanz/llvm-submodules
* Update bot: http://beanz-bot.com:8180/jenkins/job/submodule-update/


Remaining Issues
================

LNT and llvmlab will need to be updated: they rely on unique monotonically
increasing integer across branch [MatthewsRevNum]_.

Straw man Migration Plan
========================

STEP #1 : Pre Move

1. Update docs to mention the move, so people are aware of what is going on.
2. Setup a read-only version of GitHub project, mirroring our current SVN
   repository.
3. Add the required bots to implement the commit emails, as well as the umbrella
   repository update (if proposal 1 is selected) or the read-only git views for
   the sub-projects (if proposal 2 is selected).

STEP #2 : Git Move

4. Update the buildbots to pick up updates and commits from the GitHub
   repository. Not all bots have to migrate at this point, but it'll help
   provide infrastructure testing.
5. Update Phabricator to pick up commits from the GitHub repository.
6. Instruct downstream integrators to pick up commits from the GitHub
   repository.
7. Review and prepare an update for the LLVM documentation.

Until this point nothing has changed for developers, it will just
boil down to a lot of work for buildbot and other infrastructure
owners.

Once all dependencies are cleared, and all problems have been solved:

STEP #3: Write Access Move

8. Collect peoples GitHub account information, adding them to the project.
9. Switch SVN repository to read-only and allow pushes to the GitHub repository.
10. Update the documentation
11. Mirror Git to SVN.

STEP #4 : Post Move

10. Archive the SVN repository.
11. Review website links pointing to viewvc/klaus/phab etc. to point to GitHub
    instead.

.. [LattnerRevNum] Chris Lattner, http://lists.llvm.org/pipermail/llvm-dev/2011-July/041739.html
.. [TrickRevNum] Andrew Trick, http://lists.llvm.org/pipermail/llvm-dev/2011-July/041721.html
.. [JSonnRevNum] Joerg Sonnenberg, http://lists.llvm.org/pipermail/llvm-dev/2011-July/041688.html
.. [TorvaldRevNum] Linus Torvald, http://git.661346.n2.nabble.com/Git-commit-generation-numbers-td6584414.html
.. [MatthewsRevNum] Chris Matthews, http://lists.llvm.org/pipermail/cfe-dev/2016-July/049886.html
.. [submodules] Git submodules, https://git-scm.com/book/en/v2/Git-Tools-Submodules)
.. [statuschecks] GitHub status-checks, https://help.github.com/articles/about-required-status-checks/
.. [LebarCherry] Port *Cherry* to a single repository rewriting history, http://lists.llvm.org/pipermail/llvm-dev/2016-July/102787.html
.. [AminiCherry] Port *Cherry* to a single repository preserving history, http://lists.llvm.org/pipermail/llvm-dev/2016-July/102804.html
