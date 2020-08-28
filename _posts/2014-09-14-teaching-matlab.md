---
layout: post
title: "Teaching MATLAB"
---

I had the opportunity to introduce MATLAB to two groups
of 30 graduate students fitting the following profile:

1. Non-majors in Computer Science

2. Novice MATLAB programmers

3. Programming experience in _some_ language (C/Java)

Learners came in 
with a broad range of expectations:

* What is MATLAB, and what are people doing with it?,
* How to do _X_ with MATLAB?, 
* How can I use MATLAB more effectively than I already am?

I'm a huge fan of <a href="https://software-carpentry.org/" target="_blank">
Software Carpentry</a>
and their _evidence-based_ 
approach to teaching. The
argument makes sense: we are scientists, and founding our teaching
methods on hard research--or at the very least, _some_ data--is likely a
better idea than using arbitrary lesson plans and teaching techniques. 
Accordingly, as a starting point, I chose the Software Carpentry 
<a href="https://swcarpentry.github.io/python-novice-inflammation/" target="_blank">
lesson material on Python </a>
These lessons have the immense value of _feedback_ from several
workshops that have been run by trained instructors all over the world.
Of course, the first step would have had to be _translating_ all that lesson
material from Python to MATLAB. 
Which <a href="https://github.com/ashwinsrnth/bc/tree/master/novice/matlab" target="_blank">
I did</a>. This
doubled up as my very first contribution to the open-source community (yay).

The concepts covered are, in this order:

* Loading, analyzing and visualizing data from a file
* Writing MATLAB scripts and loops
* How and why to write MATLAB functions
* Conditionals in MATLAB
* Writing tests for functions

The lessons are built around a central task: analyzing and visualizing 
the data from several .csv files. With every lesson, we make our code
for doing this a little better. For example, in lesson 1, we load,
analyze and visualize the data interactively (from the command line). 
In lesson 2, we put those commands in a script, and discuss the
pros and cons over computing interactively. We proceed to introduce
loops, and modify our script to analyze _several_ datasets. And so on.

I've done sessions on MATLAB before, and I used to follow a "textbook" 
approach: exposing ideas in the same order that they would appear in
a textbook on MATLAB, for example:

* Variables and Statements
* Vectors
* Plots
* Matrices
* Loops
* Conditionals
* Scripts
* Functions

So, in one of my earlier sessions, the first few lines of code we would
type in to the command line would be something like this:

```matlab
>> a = 1
>> b = 2
>> a + b
>> a * b
>> c = [a, b]
```


Compare _that_ to the first couple of lines of code that we type in now:

```matlab
>> patient_data = csvread(`inflammation-01.csv`);
>> imagesc(patient_data)
```

I think that exposing this sort of powerful functionality early is important:
it makes learners feel like "this might actually be worth my time" and 
encourages them to participate more. 

Getting novice programmers to follow along command-by-command is slow: they're
going to meet with a lot of errors, even with the simplest of commands. The most common 
mistakes I've seen learners make in workshops:

* Typos
* Calling scripts/functions from the wrong directory

This is natural and expected; a new programming environment takes time to get
used to, and learners simply don't have enough context to make sense of 
error messages. It's tempting then, to demo-ize the whole thing and keep 
participants from writing too much code. Of course, that's a bad idea, and I prefer an
approach that's somewhere in-between:

**Commands**

Have learners type out commands on the shell while introducing ideas
for the first time and _demonstrate_ commands when expanding on them
or explaining subtleties.

**Scripts/functions**

Have learners type out stripped-down, simple versions of more complex
scripts. For example, instead of having
learners write a script that loops
over several datasets, performs analyses, and plots various figures for 
each, have them type out and execute a script that performs a single analysis
on a single dataset, and produces a single figure. _Then_ ask them to 
look at a more complex version of that script that was distributed to them
at the beginning of the session.

I also experimented with a workshop etherpad, and gave learners the option
of taking notes there instead of on their personal notepads/computers.
Most learners preferred not to interact too much with the etherpad,
I'm not sure why - maybe this should be part of the feedback - but here
are some possible reasons:

* Not enough time
* Not familiar with etherpad
* Not as convenient/useful as personal notes

I gave participants the option of providing feedback, either on the public
etherpad, or on pieces of paper. Feedback on paper was generally more
specific and comprehensive. Response was positive, with complaints
about not having enough time and not covering enough/specific material.
The structure and content of the lessons was generally appreciated,
although there were mixed opinions on the section on testing.
