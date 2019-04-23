EMTEE 1 "Version 1.0.2: Apr 2019"
=================================

[//]: # ( Convert to manpage using e.g. go-md2man -in=README.md -out=emtee.1 )

NAME
----

emtee - a faster-startup emerge -DuU --with-bdeps=y @world (et al.)

SYNOPSIS
--------

`emtee` [`-a`] [`-A`] [`-b`] [`-c`] [`-C`] [`-d`] [`-e` args] [`-E` args]
[`-h`] [`-p`] [`-N`] [`-s` set] [`-S`] [`-v`] [`-V`]

DESCRIPTION
-----------

`emtee` is a simple script to speed up *@world* updates on Gentoo
Linux systems, by significantly decreasing the time taken for the
`emerge`(1)'s initial dependency resolution stage (often the dominant
component for SBCs and other resource-challenged systems, if a binhost
is deployed).

It may be used (with appropriate options) in place of:

* `emerge --with-bdeps=y --deep --update --changed-use` *@world*,
* `emerge --with-bdeps=y --deep --update --newuse --ask --verbose` *@world*;

and similar invocations.

For example, you could achieve the same effect as the above commands
with:

* `emtee`, and
* `emtee --newuse --ask --verbose` (or, equivalently, `emtee --Nav`)

respectively.

Generally speaking, `emtee` will take materially less time
to get the first actual build underway than its counterpart `emerge
--update`, while still targeting an identical build list, permitting
job parallelism, and triggering any required slot-driven rebuilds, and
block-driven unmerges.

OPTIONS
-------

`-a`, `--ask`
  Turns on interactive mode for the real `emerge` step (i.e.
  during phase 4, as described in *ALGORITHM*, below).

`-A`, `--alert`
  Uses terminal bell notification for all interactive prompts in
  the real `emerge` step. Selecting this turns on `-a` by default.

`-b`, `--with-bdeps=y|n`
  Specifies whether or not to pull-in build time dependencies. NB,
  defaults to `y` if unspecified, in contrast to `emerge`'s default
  behaviour.

`-c`, `--crosscheck`
  Checks the final build list against the result obtained by running a
  conventional `--deep` `emerge`, and prints a `diff`(1) of the package
  list; also provides a comparitive timing, for benchmarking purposes.
  NB: does not check the *ordering* of the packages in the two lists.

`-C`, `--strict-crosscheck`
  As for `-c`, but exits with an error if the `emtee` and conventional
  `--deep --update` `emerge` lists do not match.

`-d`, `--debug`
  Prints some additional debugging information during the process.

`-e`, `--emptytree-emerge-args=`*ADDITIONAL_ARGS*
  Passes the specified arguments to the initial (`--pretend
  --emptytree`) `emerge` step used to caclulate the (unfiltered) build
  list. Note that these arguments are *not* passed to the real
  `emerge` step; you need to use `-E` for that.

`-E`, `--emerge-args=`*ADDITIONAL_ARGS*
  Passes the specified arguments to the real `emerge` step. Note that
  these arguments are *not* passed to the preliminary `emerge` step; you
  need to use `-e` for that.  
&nbsp;  
  Note also that you can achieve the effect of the `-a` `-A`, `-p` and `-v`
  options by setting them directly via `-E`, if you prefer. They are
  provided as syntactic sugar, for convenience.

`-h`, `--help`
  Prints a short help message, and exits.

`-p`, `--pretend`
  Passes the `--pretend` option to the real `emerge` step, resulting in
  a 'dry run' (nothing will actually be updated).

`-N`, `--newuse`
  Rebuild packages which have had any USE flag changes, even if those
  changes don't affect flags the user has enabled (the more
  conservative `--changed-use` behaviour is the default).

`-s`, `--set`=*SET*
  Uses the specified set (e.g. *@system*) in preference to the default
  *@world*. You can pass regular package names here as well, but note
  that using a regular `emerge` is likely to be faster for such targets.

`-S`, `--force-slot-rebuilds`
  Checks for any slot-change-driven rebuilds, and adds these (reverse)
  dependencies to the build list; this option should not generally be
  required, as the real `emerge` step will trigger such rebuilds
  automatically anyway.

`-v`, `--verbose`
  Passes the `--verbose` option to the real `emerge` step
  
`-V-`, `--version`
  Prints `emtee`'s version number, and exits.

ALGORITHM
---------

The `emtee` process runs as follows:

1. Derive a full, versioned build list for the *@world* set and its
   entire deep dependency tree, via `emerge --with-bdeps=y --pretend
   --emptytree --verbose [opts]` *@world*, which Portage can do
   relatively quickly. The resulting list, as it is derived as if *no*
   packages were installed to begin with, will automatically contain
   all necessary packages at their 'best' versions (which may entail
   upgrades, downgrades, new slots etc.  wrt the currently installed
   set).
2. Filter this list, by marking each fully-qualified atom
   (*FQA*=*$CATEGORY/$PF*) within it for building (or not). Begin
   with all *FQAs* unmarked.  
&nbsp;  
      * Then (pass 1), mark anything which isn't a block, uninstall or reinstall for build;
      * Then (pass 2), check each reinstall, to see if its *active*
        USE flag set is changing (default behaviour), or if *any* of
        its USE flags are changing (`-N`/`--newuse` behaviour), and if
        so, mark that package for build (fortunately, the `--verbose`
        output from step 1 contains the necessary USE flag delta
        information to allow us to easily work this out).
     * Then (pass 3), if `-S`/`--force-slot-rebuilds` is in use, for
       each marked package on the list whose slot or subslot is
       changing (also inferable from the phase 1 output), search
       */var/db/pkg/<FQA>/RDEPENDS* (and *DEPENDS*, if
       `--with-bdeps=y`, the default, is active) for any matching slot
       dependencies.  Mark each such located (reverse) dependency that
       is *also* on the original `--emptytree` list (and not a block
       or uninstall) for build.  
&nbsp;  
   Note that pass 3 is skipped by default, since the phase 4 emerge
   (aka the real `emerge`) will automatically trigger any
   necessary slot rebuilds anyway, so it is redundant except for in a
   few esoteric situations.
3. Iff `-c`/`--crosscheck` (or `-C`/`--strict-crosscheck`) passed,
   compare the *FQA* build list produced by invoking `emerge --bdeps=y
   --pretend --deep --update [--changed-use|--newuse] [opts]` *@world*
   (adapted for specified options appropriately), with that produced
   by invoking `emerge --oneshot --pretend [opts]`
   *<filtered-FQA-build-list-from-phase-2>*. If any differences are
   found, report them (and, additionally, stop the build in such a
   case, if `-S`/`--strict-crosscheck` specified). Also report
   a series of comparative (total elapsed wall-clock) timings for both
   alternatives, for benchmarking purposes.  
&nbsp;  
   Note: crosschecking should *only* be used for reassurance or
   benchmarking, as it will, of necessity, be slower than the baseline
   in total time cost (since the check involves running both that
   *and* the newer, `--emptytree`-based approach)! So, if your goal is
   to improve emerge times, do *not* pass `-s`/`-S`.
4. Invoke the real `emerge`, as: `emerge --oneshot [opts]`
   *<filtered-FQA-build-list-from-phase-2>*.  
&nbsp;  
   Note that additional arguments may be passed to this invocation, both
   explicitly (via `-E`/`--emerge-args`) and implicitly, via one of
   the impacting options (`-v`/`--verbose`, `-a`/`--ask`,
   `-A`/`--alert`, or `-p`/`--pretend`).

BASIS
-----

Why is this approach faster? Well, the main claims behind `emtee` are:

1. An `--emptytree` `emerge` of *@world* yields the same versioned package list
   that a `--deep --update` `emerge` would arrive at.  
&nbsp;  
   That is, for `emtee` to work, it must be true that for a consistent,
   depcleaned Gentoo system with a recently updated set of ebuild
   repositories, if `emerge --with-bdeps=y --emptytree` *@world* is
   invoked and runs successfully to conclusion, then an immediately
   following `emerge --with-bdeps=y --deep --changed-use --update`
   *@world* will always be a no-op.  
&nbsp;  
   Or, to put it another way, we claim that the list of
   fully-qualified atoms (*FQAs*, where an *FQA* is *$CATEGORY/$PF*)
   produced by running `emerge --with-bdeps=y --pretend --emptytree
   --verbose` *@world* will always describe the same end state reached
   by running `emerge --with-bdeps=y --deep --update
   [--changed-use|--newuse]` *@world* from same starting conditions,
   as regards packages and versions, anyhow.
2. It also contains sufficient information to simulate `--changed-use`
  and `--newuse`.  
&nbsp;  
   Of course, the issue is that in addition to new versions (*[N]*),
   package upgrades (*[U]*), downgrades (*[UD]*), new slots (*[NS]*)
   blocks and uninstalls, such a list will generally also contain a
   huge number of reinstalls (*[R]*). Some of these will genuinely
   need doing (in light of changed USE flags etc.), but many,
   usually the vast majority, will be redundant.  
&nbsp;  
   Fortunately, for common rebuild selections (such as `--changed-use`
   and `--newuse`), we can easily identify which is which, using only
   the information provided by the `--pretend --emptytree` `emerge`
   itself - since in its output, changes to the USE flag active set
   for a given package are shown with an _*_ suffix, and changes to
   the remaining set with a _%_ suffix, when `--verbose` is used.
3. Producing such a list, and then shallow emerging it, reduces the net
   dependency calculation time.  
&nbsp;  
   Finally, we also claim that for a Gentoo system with many installed
   packages, the time taken to 1) generate an `--emptytree` *@world*
   *FQA* list for all packages, 2) filter this to leave only those
   elements that actually *need* an install or reinstall (given the
   current package set and `--changed-use`/`--newuse`
   etc. preference); and 3) invoke a `--oneshot` `emerge` on the
   resulting list (of *=$CATEGORY/$PF* *FQAs*), to the point the first
   build actually starts, can be up to an *order of magnitude* less
   than the equivalent time to first build commencement for a `--deep
   --update` based *@world* `emerge` (for a system with many installed
   packages and where the number of required updates is (relatively)
   small).  Yet, if the other claims above are correct, the resulting
   merge lists for both approaches will be identical. Furthermore,
   this real `--oneshot` `emerge` will still deal with triggered slot
   change rebuilds and soft block uninstalls for us, and (subject to
   *EMERGE_DEFAULT_OPTS*) allow the scheduled builds to be fully
   parallelized.

ADVANTAGES
----------

The speedup for the dependency phase just mentioned, can
translate to hours saved on slow SBCs with binhost backing (where the
build phase itself is relatively low cost). The efficiency gains fall
if a large number of packages require updating, however.

Another advantage of this approach is that for some complex updates,
with many blockers, `emerge --with-bdeps=y --pretend --emptytree
--verbose` *@world* can sometimes derive a valid list of *FQAs*, in
cases where `emerge --with-bdeps=y --pretend --deep --update` *@world*
fails so to do, even with heavy backtracking (although this is a
comparatively rare situation).

Note: in the context of this script, an *FQA*, or fully qualified
atom, is taken to be *$CATEGORY/$PF*, so for example:
*sys-apps/package-a-1.0.4_rc4_p3-r2*.

BUGS
----

A number of nice `emerge` features don't work with `emtee`, such as
`--changed-deps` etc. The focus has been on `--changed-use` and
`--newuse`, which are the most common.

To operate correctly, `emtee` needs to be able to parse the output
from `emerge`. So, if the latter's format changes in the future,
expect breakage ><

The script's efficiency gains degrade rapidly as the number of
packages requiring upgrade increases.

COPYRIGHT
---------

Copyright © 2018-2019 sakaki

License GPLv3+ (GNU GPL version 3 or later)  
<http://gnu.org/licenses/gpl.html>

This is free software, you are free to change and redistribute it.  
There is NO WARRANTY, to the extent permitted by law.

AUTHOR
------

sakaki — send bug reports or comments to <sakaki@deciban.com>


SEE ALSO
--------

`diff`(1), `emerge`(1), `portage`(5)
