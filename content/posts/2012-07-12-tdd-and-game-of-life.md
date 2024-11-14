+++
author = "hannibal"
categories = ["java", "knowledge", "problem solving", "TDD"]
date = "2012-07-12"
tags = ["barcoding", "game of life"]
title = "TDD and Game of Life"
url = "/2012/07/12/tdd-and-game-of-life/"

+++

So today at 8-12PM I had a great session with two friends of mine. It was awesome. Like a mini code retreat.

We set down in a musky bar, drank wine and beer and cider, and decided to practice some TDD with the well known problem of <a href="http://en.wikipedia.org/wiki/Conway's_Game_of_Life" target="_blank">Conway's Game of Life</a>. This challenge is really interesting. I never done it before, ever. So it was a really good practice for me.

So.

**In the beginning there was Test**

One of my friends and I started out by developing the implementation for the game while the second one was mentoring and couching us. As with any problem I'm facing now days, I started with writing a failing test first. I didn't write any kind of production code yet. I wrote a test testing for having the class called game of life.

~~~java
    @Test
    public void shouldHaveClassForGameOfLife() {
        GameOfLife gameOfLife = new GameOfLife();
    }
~~~

This wasn't compiling of course because I didn't have any kind of GameOfLife class. But intelliJ is so intelligent that I simply pressed Alt+Enter and created the class immediately. The class didn't have anything in it, but I already had a passing test.

So this went on and on and I created one test after another while my other coding friend did the same.

**Now the amazing part**

I begun working on the Grid. A simple octagonal coordinating system. This was represented in the beginning with a simple two dimensional array with Cells in it.

~~~java
    Cells[][] cells = new Cells[50][50];
~~~

This of course wasn't dynamic. I didn't care about that yet. I had my grid of cells. These cells were initially all dead.

Now, the interesting part is that as I developed my Grid finding out the Cells neighbours and counting them, my friend worked on the Cells themselves and getting their next state and killing them based on the rules.

We never talked to each other, didn't agree on roles or directions or anything. And even so at the and. We were at a stage where we met in the middle and could merge our codes! Our little game of life evolved with a push of a button. ( Three actually. )

This was simply amazing. Without ever talking about the direction we want to go we created a working code base that could be merged!

**It just works**

Before TDD I would have tackled this problem much differently. And it would have taken me much more time too. This was like an hour or so.

TDD helped me break down the job into small, manageable tasks. I created and deleted and rewrote tests as I went on and on and developed the algorithm for my Grid and Cell. And eventually the problem slowly unfolded itself right before my eyes. I began to see the connections. I began to see the beauty. I began to understand! This is something I rarely enjoyed previously without using TDD.

**Summary**

I recommend for you guys to do the same. Just sit down, find a problem, find a coding kata and just do it with TDD. With PROPER TDD.

Here are some good sites for katas and problems:

<a href="http://codekata.pragprog.com/" target="_blank">http://codekata.pragprog.com/</a>

<a href="http://www.spoj.pl/problems/classical/" target="_blank">http://www.spoj.pl/problems/classical/</a>

Just select a problem and then start cracking on it. Do this every time you have some free time. Like a martial art trainee doing basic exercises and you will get better at problem solving and at TDD too. I promise.

Happy coding and good night!

Gergely.