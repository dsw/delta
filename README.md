**Maintainer:** [Daniel S. Wilkerson](http://danielwilkerson.com/)\
**Developer:** [Scott McPeak](http://www.cs.berkeley.edu/~smcpeak/)\
**Documenter:** [Simon Goldsmith](http://http.cs.berkeley.edu/~sfg/)

### Summary

Delta assists you in minimizing \"interesting\" files subject to a test
of their interestingness. A common such situation is when attempting to
isolate a small failure-inducing substring of a large input that causes
your program to exhibit a bug.

Our implementation is based on the [Delta Debugging
algorithm](http://www.st.cs.uni-sb.de/dd/).
[Andreas](http://www.whyprogramsfail.com/) wrote a book [\"Why Programs
Fail\"](http://www.whyprogramsfail.com/) about debugging programs.

This work was supported by professors [Alex
Aiken](http://theory.stanford.edu/~aiken/) and [George
Necula](http://www.cs.berkeley.edu/~necula/) and was done at UC
Berkeley.

### Releases

Feel free to just get the current Subversion repository version as a
guest user.

-   [delta-2006.08.03.tar.gz](http://delta.tigris.org/files/documents/3103/33566/delta-2006.08.03.tar.gz)
      md5: ` 7be4ac4ae9c1eb01ccf29d413d4cc64a` This is the current
    recommended release.
-   [delta-2006.07.15.tar.gz](http://delta.tigris.org/files/documents/3103/33236/delta-2006.07.15.tar.gz)
      md5: ` 57afa6d4e7d15f380803e878c24678ed` This release had a
    version that multidelta did not work without the -cpp flag.
-   [delta-2005.09.13.tar.gz](http://delta.tigris.org/files/documents/3103/25616/delta-2005.09.13.tar.gz)
      md5: ` 588d65056ea48ae2a2ecee32598c5837` This version had a bug
    that it was too hard to stop delta once it started.
-   The first release of Delta was on 14 July 2003. It is now retired.

History
-------

I first wrote delta out of necessity. Scott and I had a quarter-million
line (after prepossessing) C++ input that would crash the C++ front-end
we were working on,
[Elsa](http://www.cs.berkeley.edu/~smcpeak/elkhound/sources/elsa/);
there was just no way we were going to minimize that by hand. I just
checked my little perl script into the CVS repository most of us used in
the Programming Languages group at Berkeley. Scott later added to it.

I would hear occasionally of someone else in the group using it: just a
\"Hey, thanks for delta!\" every once in a while. One day, a graduate
student came into my office; he leaned back against the wall and spoke
in a low tone. \"You know, some of our friends at Microsoft Research
have heard of your delta tool. They would really like to use it. They
asked me to ask you if you could please *release it as Open Source*; the
BSD license is preferred.\"

I GPL\'ed it right away. A few years later I relented and it is now
released under the BSD license. So there you go, you have Microsoft to
thank for the existence of delta as an open-source project. I think it
is quite interesting given their often unfriendly public stance on Open
Source, such as that it is
[Communism](http://www.theregister.co.uk/2000/07/31/ms_ballmer_linux_is_communism/).
I guess they want to commune with us after all.

I presented Delta at [CodeCon 2006](http://www.codecon.org/2006/). The
slides are [here](delta_codecon.ppt).

Usage overview
--------------

The best way to understand how to use delta is with an example of its
usage. [Below is one example](#example) helpfully written up for me by
Simon Goldsmith; read it first. For those wanting more, I also wrote a
more detailed and harder to read document describing each tool: [Using
Delta](README_using_delta.md).

Note that what follows is an example of using delta to minimize an input
file to a program that reads programs, much as a compiler does. Note two
features of file minimization that are present in the example.

### Do a controlled experiment.

Below we don\'t just minimize a file that causes Oink to produce an
error message, we minimize a file that causes gcc to accept AND oink to
reject in a specific way. That is, the test delta does is a controlled
experiment, where gcc is the control. Ignoring this aspect of the
problem seems to be a frequent mistake of first time users.

### Exploit nested structure.

One may minimize files of simpler syntax than C++ but really all files
are interesting in the first place because they are in some language or
another. Some simple configuration files are literally just a list of
lines but most languages have some nested structure. Multidelta filters
the input through the topformflat utility (included) to suppress any
newlines past a particular nesting depth; this \"explains\" the nesting
structure to the otherwise line-oriented delta utility (a brilliantly
simple idea of Scott McPeak\'s). If your input file language has no
nesting structure, you can hack on multidelta to remove the filtration
through topformflat or just use the raw delta program. If your language
has a different nesting structure than C/C++, you can write your own
multidelta and substitute it. A simple flex program should suffice; it
need not be terribly accurate for delta to do well. []{#example}

An example
----------

Simon Goldsmith 8 April / 12 Sept, 2005.

Note that this example is edited for simplicity from the raw output; we
sincerely hope we did not introduce any bugs.

### Setup

\(1) Make a new directory and copy the file to be minimized there.

    % mkdir deltaexample
    % cd deltaexample/
    % cp ../nsCSSDataBlock-23801-1112390043.cpp.g.ii ./foo.ii
    % chmod +w foo.ii

\(2) (optional) Put a read-only backup copy of the file in, say, orig/ .

    % mkdir orig
    % cp foo.ii orig/
    % chmod -R a-w orig

### Define interestingness

\(3) Write a script (do not call it \'test\' as that is a system utility
program) to test the interestingness of the file, as we do below.

Note that for this example, \"interesting\" means the file passes gcc
but fails oink with a particular error message. That is, if 1) gcc
accepts, and 2) oink rejects with the desired error message, then we
return zero (meaning \"interesting\"). If anything else happens then we
return a nonzero exit code (meaning \"not interesting\")

Some reminders about shell: a zero exit code means \"true\"; so for the
purposes of &&, a zero exit code means \"keep going\" and grep returns 0
if it matches, nonzero if not. We redirect output to /dev/null because
the output of delta is noisy. Be careful of quoting hell: notice that
we\'ve used \'.\' to match characters like single quote.

    % cat > test1.sh
    #!/bin/bash

    FILE=foo.ii
    OINK=/home/simon/oink_all/oink/oink
    GCC=/usr/bin/gcc

    $GCC -c $FILE -o /dev/null &> /dev/null && $OINK $FILE | \
      grep 'error: cannot convert argument type .class .* const &. ' \
      'to receiver parameter type' \
      &> /dev/null
    ^D

\(4) Make the script executable and run it on the file \-- make sure it
returns 0. Optionally turn off the redirection to /dev/null temporarily
to check the error message that is being found by the grep.

    % chmod +x test1.sh
    % ./test1.sh foo.ii ; echo $?
    0

### Minimize automatically

\(5) Run multidelta with the script on the file several times at, say,
levels 0 0 1 1 2 2 10.

    % multidelta -level=0 ./test1.sh foo.ii
    (check email)
    % multidelta -level=0 ./test1.sh foo.ii
    (read slashdot)
    % multidelta -level=1 ./test1.sh foo.ii
    % multidelta -level=1 ./test1.sh foo.ii
    % multidelta -level=2 ./test1.sh foo.ii
    % multidelta -level=2 ./test1.sh foo.ii
    % multidelta -level=10 ./test1.sh foo.ii
    % multidelta -level=10 ./test1.sh foo.ii

\(6) The input file will be modified in place and you should be left with
something smaller.

    [simon@otter][deltaexample]$ ls -l
    total 116
    -rw-r--r--  1 simon simon  8451 Sep 12 17:10 foo.ii
    -rw-r--r--  1 simon simon  8948 Sep 12 17:10 foo.ii.bak
    -rw-r--r--  1 simon simon  8451 Sep 12 17:10 foo.ii.ok
    -rw-r--r--  1 simon simon 57739 Sep 12 17:10 log
    -rw-r--r--  1 simon simon  2752 Sep 12 17:10 multidelta.log
    dr-xr-xr-x  2 simon simon  4096 Sep 12 17:16 orig/
    -rwxr-xr-x  1 simon simon   385 Sep 12 16:36 test1.sh*
    -rw-r--r--  1 simon simon    11 Sep 12 16:02 test1.sh~

    [simon@otter][deltaexample]$ ls -l orig/
    total 552
    -r--r--r--  1 simon simon 558970 Sep 12 16:00 foo.ii

### Minimize further by hand

\(7) Hack on foo.ii by hand, re-running test1.sh each time to check it is
still \"interesting\". Sometimes it helps to hack on foo.ii a little to
get delta unstuck and then rerun delta again. You might want to run
indent as well whenever you stop to look at foo.ii as topformflat makes
a mess.

Final file:

    class A {};
    int main() {
      const A *val;
      val->~A ();
    }

Note that the original file was about 560 KB!

Endorsements
------------

This section is just for fun because I\'ve never had a tool so widely
used before.

    Subject: Thanks for Delta
       Date: Thu, 21 Jul 2005 21:13:20 -0700
       From: Flash Sheridan
         To: Daniel S. Wilkerson

    This is just a quick thank-you note for Delta.  Andrew Pinski pointed
    me towards it after filing a GCC bug with a very long source file; it
    immediately reduced a different bug file from 16K lines to ten (GCC
    bug 22604).  Oddly enough, it initially found a different bug (22603),
    since I'd only specified "internal compiler error", not "segmentation
    fault".  I might not have been able to file either of these bugs
    without Delta, since the code was proprietary, but the two
    Delta-reduced files were small enough to make public.


    Subject: Re: Thanks for Delta
       Date: Tue, 13 Sep 2005 10:56:10 -0700
       From: Flash Sheridan
         To: Daniel S. Wilkerson

    Delta has become even more valuable since my initial thank-you note.
    I'm not sure it's helped with all of the GCC bugs I've been filing
    (I've been tracking them at
    http://pobox.com/~flash/FlashsOpenSourceBugReports.html), but I
    couldn't have filed most of them without Delta.  Typically I find a
    bug when GCC is compiling a large, confidential file, which I couldn't
    post to Bugzilla.  Delta has always been able to find a radically
    smaller file, which I have been able to attach to my bug report.


    Date: Sun, 23 Oct 2005 22:01:42 +0200 (CEST)
    From: Richard Guenther
      To: Daniel S. Wilkerson

    > > > Thanks for your interest in Delta.  I would be interested to
    > > > hear more about what you are doing with it.  If it is something
    > > > I can put in the endoresements section ("Delta saved my
    > > > daugher's life!") that would be great.
    > >
    > > Well, delta is saving a lot of gcc developers life ;) I would
    > > guess 1 of 3 bugs sumitted to the gcc bugzilla get their testcase
    > > reduced using delta.
    > 
    > Holy moly!  Can I quote that publicly such as on my web page?

    Yes - a little bit more accurate would be to say we're using delta to
    reduce all testcases from the gcc bugzilla in case they get entered
    unreduced.

------------------------------------------------------------------------

Delta (both the algorithm and this tool) has been used in the [Cal
Berkeley
(CS169)](http://www-inst.eecs.berkeley.edu/~cs169/fa05/index.shtml) and
[Stanford (CS295)](http://www.stanford.edu/class/cs295/) in software
engineering classes.

    Subject: Re: delta debugging on instructional machines?
       Date: Mon, 24 Oct 2005 22:55:08 -0700
       From: Gilad Arnold 
         To: Daniel S. Wilkerson

    We've just assigned a delta-related homework to the students today,
    . . .  We do hope that we can actually convince the students to use
    delta in the course of their project development, but time will
    tell. And thanks again!


    Subject: Re: use of delta in your class
       Date: Mon, 24 Oct 2005 13:57:22 -0700
       From: Alex Aiken
         To: Daniel S. Wilkerson

    Yes, I gave them a homework assignment for CS295 using delta.  Feedback 
    was positive but unquantified.

 
-
