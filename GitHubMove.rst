==============================
Moving LLVM Projects to GitHub
==============================

Introduction
============

..
  TODO: Should be consistent wrt "sub-project" or "subproject".

..
  TODO: Should be consistent wrt capitlization of "git".  git's official
  documentation uses upper-case 'G', but most other people I've seen use
  lower-case.

This is a proposal to move our current revision control system from our own
hosted Subversion to GitHub. Below are the financial and technical arguments as
to why we need such a move and how will people (and validation infrastructure)
continue to work with a Git-based LLVM.

There will be a survey pointing at this document which we'll use to gague the
community's reaction and, if we collectively decide to move, the time-frame. Be
sure to make your view count.

This proposal is divided into the following parts:

* Outline of the reasons to move to Git and GitHub
* Description on the options
* What some examples of workflow will look like (compared to currently)
* The proposed migration plan

What This Proposal is *Not* About
=================================

The development of LLVM will continue as it exists now, with the same policies.

This proposal relates only to moving the hosting of our source-code repository
from SVN hosted on our own servers to git hosted on GitHub. We are not proposing
other workflow changes here.  That is, it should not be assumed that moving to
GitHub implies using GitHub's issue tracking, or using the GitHub UI for
pull-requests and/or code-review.

Every existing contributors will get commit access on demand. Those who don't
have an existing GitHub account will have to create one in order to continue
having commit access.

Why Git, and Why GitHub?
========================

Why Move At All?
----------------

The strongest reason for the move, and why this discussion started in the first
place, is that we currently host our own Subversion server and Git mirror in a
voluntary basis. The LLVM Foundation sponsors the server and provides limited
support, but there is only so much it can do.

Volunteers are not sysadmins themselves, but compiler engineers that happen
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

..
  TODO: "'multiple' LLVM developers" should be strengthened.  Do we have any
  evidence for 'most'?  Rewritten using what data we do have, but as-is is not
  as strong as can be, I think.  I don't know if this is important -- depends
  on how much resistance there is to git vs svn.

Git is also the version control many (most?) LLVM developers use. Despite the
sources being stored in a SVN server, these developers are already using git
through the Git-SVN integration.

Git allows you to:

* Commit, squash, merge, and fork locally without touching the remote server.
* Maintain as many local branches as you like, letting you maintain multiple
  threads of development.
* Collaborate on these branches (e.g. through your own fork of llvm on github).
* Inspect the repository history (blame, log, bisect) without Internet access.

In addition, because Git seems to be replacing most OSS projects' version
control systems, there are many tools that are built over Git. Future tooling is
much more likely to support Git first (if not only).

Why GitHub?
-----------

..
  Note: Since LLVM is primarily an American project, we should probably use the
  American convention of referring to corporations as singular ("GitHub
  provides," rather than "GitHub provide").

GitHub, like GitLab and BitBucket, provides free code hosting for open source
projects. Any of these could replace the code-hosting infrastructure that we
have today.

These services also have a dedicated team to monitor, migrate, improve and
distribute the contents of the repositories depending on region and load.

All things being equal, GitHub has one important advantage over GitLab and
BitBucket: It offers read-write **SVN** access to the repository
(https://github.com/blog/626-announcing-svn-support).
This would enable people to continue working post-migration as though our code
were still canonically in an SVN repository.

In addition, there are already multiple LLVM mirrors on GitHub, indicating that
part of our community has already settled there.

On Managing Revision Numbers with Git
-------------------------------------

The current SVN repository hosts all the LLVM sub-projects alongside each other.
A single revision number (e.g. r123456) thus identifies a consistent version of
all LLVM subprojects.

Git does not use sequential integer revision number but instead uses a hash to
identify each commit. (Linus mentioned that the lack of such revision number
is "the only real design mistake" in git [TorvaldRevNum]_.)

The loss of a sequential integer revision number has been a sticking point in
past discussions about git:

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

However, git can emulate this increasing revision number: `git rev-list  --count
<commit-hash>`. This identifier is unique only within a single branch, but this
means the tuple `(num, branch-name)` uniquely identifies a commit.

We can thus use this revision number to ensure that e.g. `clang -v` reports a
user-friendly revision number (e.g. `master-12345` or `4.0-5321`). This should
be enough to address the objections raised above with respect to this aspect of
git.

What About Branches and Merges?
-------------------------------

In contrast to SVN, Git makes branching easy. Git's commit history is represented
as a DAG, a departure from SVN's linear history.

However, we propose to *enforce linear history* in our canonical git repository
repository.  (This is not uncommon amongst many large users of git.)

..
  TODO: Is this going to work when people push via the SVN bridge?

We'll do this with a combination of client-side and server-side hooks. GitHub
offers a feature called `Status Checks`: a branch protected by `status checks`
requires commits to be whitelisted before the push can happen.  A supplied
pre-push hook on the client side will run and check the history, before
whitelisting the commit being pushed [statuschecks]_.

What About Commit Emails?
-------------------------

An extra bot will need to be set up to continue to send emails for every commit.
We'll keep the exact same email format as we currently have (a change is possible
later, but beyond the scope of the current discussion), the only difference
being changing the URL from `http://llvm.org/viewvc/...` to
`http://github.org/llvm/...`.


One or Multiple Repositories?
=============================

There are two major proposals for how to structure our git repository: The
"multirepo" and the "monorepo".

1. *Multirepo* - Moving each SVN sub-project into its own separate git repository.
2. *Monorepo* - Moving all the LLVM sub-projects into a single git repository.

The first proposal would mimic the existing official separate read-only git
repositories (e.g. http://llvm.org/git/compiler-rt.git), while the second one
would mimic an export of the SVN repository (i.e. it would look similar to
https://github.com/llvm-project/llvm-project, where each sub-project has its own
top-level directory).

Why monorepo?
-------------

Full disclosure, the authors of this document are in favor of the monorepo.  :)

First, the monorepo is carefully designed to minimize workflow disruptions to
those who don't want to make a change.

Under the monorepo approach, we would continue to maintain the existing
read-only git mirrors for each subproject (e.g.
http://llvm.org/git/compiler-rt.git).  Users will be able to push from these
into the monorepo using git-svn (the push target would be GitHub's svn endpoint
for the monorepo), so developers who want to continue using the existing git
mirrors as they do today would have minimal workflow disruption.  It's just a
matter of updating your git-svn configs.
n
Similarly, users who are interested in only one or a few subprojects could
continue to work with the individual single-subproject mirrors as they do today,
rather than being forced to download history for all LLVM subprojects.

Those who wish to use SVN could also use the monorepo via GitHub's git-to-svn
bridge.  Revision numbers and maybe the directory structure would change, but
beyond that it would continue to work as normal.  In contrast, SVN users would
be more disrupted in the multirepo approach.  They could have a separate SVN
repository for each sub-project, but under this proposal we currently have no
way to let them synchronize their separate repositories to a consistent state
like `svn checkout r123456` currently does.

Of course nobody will be forced to compile projects they don't want to build.
The exact structure is TBD, but even if you use the monorepo directly, we'll
ensure that it's easy to set up your build to compile only a few particular
subprojects.

But none of this answers the question of, what are the *advantages* of the
monorepo?

For one thing, many of us would prefer the workflow the monorepo affords.
There's lots more on that below, but at a high level, many operations that
affect multiple subprojects are complex or impossible in the multirepo, but they
become dead-simple in the monorepo.  For example, in the multirepo, you can't
atomically commit across multiple subprojects in the multirepo, and maintaining
a proper branch with interleaved commits, some in one subproject and some in
another, requires lots of git submodule hackery.  In contrast, these are just
"git commit" and regular git branch operations in the monorepo; no magic
required.

Even the common operation of checking out compatible versions of LLVM and (say)
clang is far simpler in the monorepo.  More on this below, too.

But the monorepo has less immediate/technical advantages over the multirepo as
well.  Because all of our subprojects would live in a single repository, we
could move code between them easily and without losing history.  This would let
us, for example, reuse a data structure from LLDB within LLVM by moving it to
libSupport.  It would also let us extract some pieces of libSupport and ADT to a
new top-level, independent library that could be reused by e.g. libcxxabi.

Finally, the monorepo would make it easier for developers to update all
subprojects when changing an API or refactoring code (e.g. `git grep` would work
across all subprojects).

How Do We Handle A Single Revision Number Across Multiple Repositories?
-----------------------------------------------------------------------

A key need is to be able to check out multiple projects (i.e. lldb+llvm or
clang+llvm+libcxx for example) at a specific revision.

Under the monorepo, this is a non-issue.  That proposal maintains property of
the existing SVN repository that the sub-projects move synchronously, and a
single revision number (or commit hash) identifies the state of the development
across all projects.

Under the multirepo, things are more involved.  We describe here the proposed
solution.

Fundamentally, separated git repositories imply that a tuple of revisions
(one entry per repository) is needed to describe the state across
repositories/sub-projects.
For example, a given version of clang would be
*<LLVM-12345, clang-5432, libcxx-123, etc.>*.

To make this more convenient, a separate *umbrella* repository would be
provided. This repository would be used for the sole purpose of understanding
the sequence (with some granularity) in which commits were added across
repository and to provide a single revision number.

This umbrella repository will be read-only and periodically updated
to record the above tuple. The proposed form to record this is to use git
[submodules]_, possibly along with a set of scripts to help check out a
specific revision of the LLVM distribution.

A regular LLVM developer does not need to interact with the umbrella repository
-- the individual repositories can be checked out independently -- but you would
need to use the umbrella repository to bisect or to check out old revisions of
llvm plus another subproject at a consistent version.

One example of such a repository is Takumi's llvm-project-submodule
(https://github.com/chapuni/llvm-project-submodule).  You can use `git submodule
init` to check out only the subprojects you're interested in, and other
submodule commands to e.g. update all submodules to an older revision.

This umbrella repository will be updated automatically by a bot (running on
notice from a webhook on every push, and periodically). Note that commits in
different repositories pushed within the same time frame may be visible together
or in undefined order in the umbrella repository.

Workflow Before/After
=====================

This section goes through a few examples of workflows.

Checkout/Clone a Single Project, without Commit Access
------------------------------------------------------

Except the URL, nothing changes. The possibilities today are::

  svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
  # or with git
  git clone http://llvm.org/git/llvm.git

After the move to GitHub, you would do either::

  git clone https://github.com/llvm-project/llvm.git
  # or using the GitHub svn native bridge
  svn co https://github.com/llvm-project/llvm/trunk

The above works for both the monorepo and the multirepo, as we'll maintain the
existing read-only views of the individual subprojects.

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

Commits are performed using `svn commit` or `git commit` and `git svn dcommit`.

Multirepo Proposal
^^^^^^^^^^^^^^^^^^

With the multirepo proposal, nothing changes but the URL, and commits can be
performed using `svn commit` or `git commit` and `git push`::

  git clone https://github.com/llvm/llvm.git llvm
  # or using the GitHub svn native bridge
  svn co https://github.com/llvm/llvm/trunk/ llvm

Monorepo Proposal
^^^^^^^^^^^^^^^^^

With the monorepo, there are multiple possibilities to achieve this.  First,
you could just clone the full repository::

  git clone https://github.com/llvm/llvm-projects.git llvm
  # or using the GitHub svn native bridge
  svn co https://github.com/llvm/llvm-projects/trunk/ llvm

At this point you have every sub-project (llvm, clang, lld, lldb, ...), which
**doesn't imply you have to build all of them**. You can still build only
compiler-rt for instance. In this way it's not different from someone who would
check out all the projects with SVN today.

You can commit as normal using `git commit` and `git push` or `svn commit`, and
read the history for a single project (`git log libcxx` for example).

If you don't want to have the sources for all the sub-projects checked out for,
there are again a few options.

First, you could hide the other directories using a git sparse checkout::

  git config core.sparseCheckout true
  echo /compiler-rt > .git/info/sparse-checkout
  git read-tree -mu HEAD

The data for all subprojects is still in your `.git` directory, but in your
checkout, you only see `libcxx`.  Git compresses its history very well, so a
clone of everything is only about 2x as much data as a clone of llvm only (and
in any case this is dwarfed by the size of e.g. an llvm objdir).

Before you push, you'll need to fetch and rebase as normal.  However when you
fetch you'll likely pull in changes to subprojects you don't care about.  You
may need to rebuild and retest, but only if the fetch included changes to a
subproject that your change depends on.  You can check this by running::

  git log origin/master@{1}..origin/master libcxx

..
  TODO: Do we need the second "origin/master" above?  Would be cool if the
  command was even shorter.

This shows you all of the changes to `libcxx` since you last fetched.  (This is
an extra step that you don't need in the multirepo, but for those of us who
work on a subproject that depends on llvm, it has the advantage that we can
check whether we pulled in any changes to say clang *or* llvm.)

A second option is to use svn via the GitHub svn native bridge::

  svn co https://github.com/llvm/llvm-projects/trunk/compiler-rt compiler-rt  —username=...

This checks out only compiler-rt and provides commit access using "svn commit",
in the same way as it would do today.

Finally, you could use *git-svn* and one of the subproject mirrors::

  # Clone from the single read-only git repo
  git clone http://llvm.org/git/llvm.git
  cd llvm
  # Configure the SVN remote and initialize the svn metadata
  $ git svn init https://github.com/joker-eph/llvm-project/trunk/llvm —username=...
  git config svn-remote.svn.fetch :refs/remotes/origin/master
  git svn rebase -l

In this case the repository contains only a single subproject, and commits can
be made using `git svn dcommit`, again **exactly as we do today**.

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

Multirepo Proposal
^^^^^^^^^^^^^^^^^^

With the multirepo proposal, the umbrella repository enters the dance. This is
where the mapping from a single revision number to the individual repositories
revisions is stored.::

  git clone https://github.com/llvm-beanz/llvm-submodules
  cd llvm-submodules
  git checkout $REVISION
  git submodule init
  git submodule update clang llvm libcxx

At this point the clang, llvm, and libcxx individual repositories are cloned
and stored alongside each other. There exist flags you can use to inform CMake
of your directory structure, and alternatively you can just symlink `clang` to
`llvm/tools/clang`, etc.

Monorepo Proposal
^^^^^^^^^^^^^^^^^

The repository contains natively the source for every sub-projects at the right
revision, which makes this straightforward::

  git clone https://github.com/llvm/llvm-projects.git llvm
  cd llvm
  git checkout $REVISION

As before, at this point clang, llvm, and libcxx are stored in directories
alongside each other.

Commit an API Change in LLVM and Update the Sub-projects
--------------------------------------------------------

Today this is easy for subversion users, and possible but very complicated for
git-svn users.  Most git users don't try to e.g. update LLD or Clang in the
same commit as they change an LLVM API.

The multirepo proposal does not address this: one would have to commit and push
separately in every individual repository. It might be possible to establish a
protocol whereby users add a special token to their commit messages that causes
the umbrella repo's updater bot to group all of them into a single revision.

The single repository proposal handles this natively and makes this use case
trivial.

Branching/Stashing/Updating for Local Development or Experiments
----------------------------------------------------------------

Currently
^^^^^^^^^

SVN does not allow this use case, but developers that are currently using
git-svn can do it. Let's look in practice what it means when dealing with
multiple sub-projects.

To update the repository to tip of trunk::

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

Multirepo Proposal
^^^^^^^^^^^^^^^^^^

The multirepo works the same as the current git workflow: every command needs
to be applied to each of the individual repositories.

Monorepo Proposal
^^^^^^^^^^^^^^^^^

Regular git commands are sufficient, because everything is in a single
repository:

To update the repository to tip of trunk::

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

Under either the multirepo or the monorepo, downstream projects can continue
working pretty much the same as they currently do, under either the monorepo or
multirepo proposal.

* If you were pulling from the SVN repo before the switch to git, you can
  continue to use SVN. The main caveat is that you'll need to be prepared for a
  one-time change to the revision numbers.

* If you were pulling from one of the existing git repos, this also will
  continue to work as before.

Under the monorepo proposal, you have a third option: migrating your fork to
the monorepo.  This can be particularly beneficial if your fork touches
multiple subprojects (e.g. llvm and clang), because now you can commingle
commits to llvm and clang in a single repository.

As a demonstration, we've migrated the "Cherry" fork to the monorepo in two ways:

* Using a script that rewrites history (including merges) so that it looks like
  the fork always lived in the monorepo [LebarCherry]_.  The upside of this is
  when you check out an old revision, you get a copy of all llvm subprojects at
  a consistent revision.  (For instance, if it's a clang fork, when you check
  out an old revision you'll get a consistent version of llvm proper.)  The
  downside is that this changes the fork's commit hashes.

* Merging the fork into the monorepo [AminiCherry]_.  This preserves the fork's
  commit hashes, but when you check out an old commit you only get the one
  subproject.

..
  FIXME: more details? For example how to upstream internal patches?

Monorepo Variant
================

A variant of the monorepo proposal is to group together in a single repository
only the projects that are *rev-locked* to LLVM (clang, lld, lldb, ...) and
leave projects like libcxx and compiler-rt in their own individual and separate
repository.

In this configuration, the monorep might include libcxx, compiler-rt, etc., as
submodules, or it might not.

The authors of this proposal think that this variant is less useful than a
monorepo that contains everything:

* The cost to users of the monorepo of its containing libcxx, compiler-rt, etc.
  is very small (they're tiny projects compared to llvm proper), and many users of
  the monorepo would benefit from having all of the pieces needed for a full
  toolchain present in one repository.

* Developers who hack only on one of these subprojects can continue to use the
  single subproject git mirrors, so their workflow is unchanged.  (That is,
  they aren't forced to download or check out all of llvm, clang, etc. just to
  make a change to libcxx.)

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

STEP #1 : Before The Move

1. Update docs to mention the move, so people are aware of what is going on.
2. Set up a read-only version of the GitHub project, mirroring our current SVN
   repository.
3. Add the required bots to implement the commit emails, as well as the
   umbrella repository update (if the multirepo is selected) or the read-only
   git views for the sub-projects (if the monorepo is selected).

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

8. Collect developers' GitHub account information, and add them to the project.
9. Switch the SVN repository to read-only and allow pushes to the GitHub repository.
10. Update the documentation
11. Mirror Git to SVN.

STEP #4 : Post Move

10. Archive the SVN repository.
11. Update links on the LLVM website pointing to viewvc/klaus/phab etc. to
    point to GitHub instead.

.. [LattnerRevNum] Chris Lattner, http://lists.llvm.org/pipermail/llvm-dev/2011-July/041739.html
.. [TrickRevNum] Andrew Trick, http://lists.llvm.org/pipermail/llvm-dev/2011-July/041721.html
.. [JSonnRevNum] Joerg Sonnenberg, http://lists.llvm.org/pipermail/llvm-dev/2011-July/041688.html
.. [TorvaldRevNum] Linus Torvald, http://git.661346.n2.nabble.com/Git-commit-generation-numbers-td6584414.html
.. [MatthewsRevNum] Chris Matthews, http://lists.llvm.org/pipermail/cfe-dev/2016-July/049886.html
.. [submodules] Git submodules, https://git-scm.com/book/en/v2/Git-Tools-Submodules)
.. [statuschecks] GitHub status-checks, https://help.github.com/articles/about-required-status-checks/
.. [LebarCherry] Port *Cherry* to a single repository rewriting history, http://lists.llvm.org/pipermail/llvm-dev/2016-July/102787.html
.. [AminiCherry] Port *Cherry* to a single repository preserving history, http://lists.llvm.org/pipermail/llvm-dev/2016-July/102804.html
