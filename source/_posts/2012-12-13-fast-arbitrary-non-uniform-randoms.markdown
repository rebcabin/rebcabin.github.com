---
layout: post
title: "Fast Arbitrary Non-Uniform Randoms"
author: "Brian Beckman"
date: 2012-12-13 08:28
comments: true
categories: [Randoms, PseudoRandoms, Algorithms, Pseudo Random Number Generation, Random Number Generation]
---
You give me an array of integers representing the distribution of
outcomes of some discrete random process (translation: you give me the
specification of a loaded die: "I want 1 to come up 7/39 of the time;
2 to come up 5/39 of the time; I'd better NEVER see a 3 'cause I'm
betting the farm on that; 4 to come up 11/39 of the time; 5 to come up
3/39 of the time; and 6 to come up 13/39 of the time.").  You also
give me a uniform pseudo-random number generator (PRNG) like
Mathematica's [Random
Integer](http://reference.wolfram.com/mathematica/ref/RandomInteger.html?q=RandomInteger&lang=en).
I give you back a new PRNG that honors the distribution you gave me,
in the sense that, statistically, it generates outcomes with the same
distribution as you gave me.

The problem is a practical, real-world problem.  Any time you need to
generate random numbers according to some arbitrary, given
distribution such as in simulation, traversing a Bayesian network, a
decision tree, a packet retry backoff schedule, you name it, the
problem comes up.  In my work, it comes up every time I need to
generate random expressions for testing parsers or for Monte-Carlo
search for formulas.  Most of the time, you want to generate
expressions non-uniformly.  For instance, you'd like to generate
addition expressions more frequently than division expressions or
function calls more frequently than function definitions.

There are a lot of solutions to this problem.  Probability mavens will
immediately see that the problem boils down to [inverting the
CDF](http://en.wikipedia.org/wiki/Inverse_transform_sampling).  We can
do this by:

* linear search -- O(N) space, O(N) time, where N is the number of different outcomes -- the number of elements in the original array -- the number of sides on the loaded die -- 6 in our example
* binary search -- O(N) space, O(log(N)) time -- this works because the CDF is monotonically increasing
* by constructing an explicit inverse in an array -- O(S) space, O(1) time, where S is the sum of the numbers in the array -- the total number of trials imputed in the original distribution -- 39 in our example 
* by my favorite method: [Walker's method of aliases](http://code.activestate.com/recipes/576564-walkers-alias-method-for-random-objects-with-diffe/) -- O(N) space and O(1) time.

At first glance, this seems ridiculous, but there is a way.  In a
nutshell: paint a dartboard with colors in the given proportions,
throw darts, lookup the colors (thanks to Terence Spies for this
beautiful idea).

Less metaphorically: imagine that the counts are colored balls distributed in bins:

1. 7 balls of color number 1 in bin number 1
2. 5 balls of color number 2 in bin number 2
3. 0 balls of color number 3 in bin number 3
4. 11 balls of color number 4 in bin number 4
5. 3 balls of color number 5 in bin number 5
6. 13 balls of color number six in bin number 6

Scale up the given counts of balls until the total is divisible by N
-- multiply every count by D/S where D=lcm(N,S)=lcm(6,39)=78: D/S=2

Now we have

1. 14 balls of color number 1 in bin number 1
2. 10 balls of color number 2 in bin number 2
3. 0 balls of color number 3 in bin number 3
4. 22 balls of color number 4 in bin number 4
5. 6 balls of color number 5 in bin number 5
6. 26 balls of color number six in bin number 6

Redistribute the counts amongst the bins so that every bin contains
the same number of counts, namely D/N=13, and that the following
conditions obtain:

* every new bin contains at most 2 colors
* you never take ALL the balls out of any bin -- every bin contains at least one ball of its original color if it had any

There is a very elegant algorithm for this redistribution process.
It's a one-liner whose statement is a proof of its correctness.  Think
a minute, if you like, to see if you can come up with it, but here it
is:

Fill up the shortest bin from the tallest bin; remove the new filled
bin from the game and continue.

Do this procedure on a piece of paper:

1. Take 13 balls from bin 6 leaving 13 balls; fill up and remove bin 3
2. Take 7 balls from bin 4 leaving 15 balls; fill up and remove bin 5
3. Take 3 balls from bin 4 leaving 12 balls; fill up and remove bin 2
4. Take 1 ball from bin 1 leaving 13 balls; fill up and remove bin 4
5. Bins 1 and 6 are left with 13 balls each; we're done (mechanically, you can imagine taking 0 balls from bin 6, filling up bin 1; removing bin 1, leaving just one bin -- bin 6).

you should end up with 

1. 13 balls of color 1 in bin 1 and no balls of any foreign color
2. 10 balls of color 2 and 3 balls of color 4 in bin 2
3. 0 balls of color 3 and 13 balls of color 6 in bin 3
4. 12 balls of color 4 and 1 balls of color 1 in bin 4
5. 6 balls of color 5 and 7 balls of color 4 in bin 5
6. 13 balls of color 6 in bin 6 and no balls of any foreign color

Now, to generate new randoms, roll once to choose a bin
`RandomInteger[{1,N}]`, and roll again to choose a height
`RandomInteger[{1,D/N}]`.  If the randomly chosen height is less than
or equal to the number of balls of the home color, return the number
of the home color; otherwise, return the number of the foreign color.

Nifty, eh?  Here is a Mathematica notebook with an implementation
tuned for clarity:
[https://dl.dropbox.com/u/1997638/FastNonUniformPseudoRandoms.cdf](https://dl.dropbox.com/u/1997638/FastNonUniformPseudoRandoms.cdf)