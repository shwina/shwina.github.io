---
layout: post
title: "Profiling Parallel Programs"
---

In high performance computing,
performance is kind of a big deal.
And the first step in performance analysis 
and performance *improvement*
is
[profiling](https://en.wikipedia.org/wiki/Profiling_%28computer_programming%29).

High performance computing almost always entails some
form of [parallelism](https://en.wikipedia.org/wiki/Parallel_computing).
And parallel programs are plain hard. They're harder to write,
harder to debug, and harder to profile.

# gprof

`gprof` is pretty great. Just compile your code with `-pg`, and `-g`,

```bash
$ gcc -pg -g -O0 hello.c bye.c -o hibye.exe
```

run your code as usual,

```
$ ./hibye.exe
```

and you'll see `gmon.out`. Now,

```
$ gprof hibye.exe gmon.out
```

should summarize the performance of your code.
Beware, `gprof` will not
pick up on any calls to shared library functions.
OK, that's a downer, and
there's lots more. But it's easy to use, and gives me quick results.
With the legacy code I work with, where there *are* no shared library calls,
`gprof` is pretty awesome.

# gprof + MPI

`gprof` isn't designed to work with MPI code.
But, as is generally the case with these things,
it's possible with sufficient abuse:

First, set the environment variable `GMON_OUT_PREFIX`:

```bash
$ export GMON_OUT_PREFIX=gmon.out-
```

Then, the usual business:

```bash
$ mpicc -pg -g -O0 hello.c bye.c -o hibye.exe
$ mpiexec -n 32 hibye.exe
```

You should see 32 (or however many processes) files,
with names `gmon.out-<pid>`.
This is an undocumented feature of `glibc`,
and it really shouldn't be - it's massively useful.

Now you have a separate `gmon.out` file for every
MPI process. Awesome. Sum them:

```bash
$ gprof -s hibye.exe gmon.out-*
```

And use the resulting `gmon.sum` to generate
`gprof` output:

```bash
$ gprof hibye.exe gmon.sum
```

[Credit](https://cluster.earlham.edu/wiki/index.php/Cluster:Gprof#Basic_Recipe_-_Parallel_MPI_Code)
where it's due. 
Now, I haven't figured out how to replace the `pid`
with the MPI rank - 
this could be exponentially more useful to some users.
And the method mentioned in the source doesn't really
seem to be working.
But I'm sure this is possible with some ingenuity.

# mpiP

[mpiP](https://mpip.sourceforge.net/) is a neat little
tool for profiling MPI applications.
In particular, it's extremely useful in figuring out
how much your application is spending time *communicating*
relative to *computing*.

The documentation for setting up and using `mpiP`
is complete (good), but small (better).
Once you have `mpiP` set up, profiling your code is
as easy as linking it with the `mpiP` library and some
other stuff it needs:

```bash
$ mpicc -g -O0 hello.c bye.c -o hibye.exe -lmpiP -liberty -lbfd -lunwind
```

Running your code (`mpiexec`) will produce `mpiP` output.

I've found that while `gprof` and `mpiP` are great tools
that do different things, using them *both* gives
me a very good idea of where my programs are spending time
and where I should focus optimization efforts.
