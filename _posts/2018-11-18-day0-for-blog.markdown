---
layout: post
title:  "Day0 for this site"
date:   2018-11-18 12:30:00 +0200
tags: random
author: ikke
---


# Day0 for this site, Welcome!

Last Friday we had a hack day amongst our Red Hat Nordics SA colleagues.
During such hackdays we spend time together to enhance our demos,
and nowadays specificly our Demo LAB. Yes, we lucky bastards got our
own demo lab :D, woot woot! I'll start our blog series by describing
how I created this web site.

We'll provide you, dear reader, samples of automation and configs
here. So we decided to create Github organization for it. That is
the place we'll use to share git repositories with samples. It's
available here: https://github.com/RedHatNordicsSA

## Finding the site

<img src="https://assets-cdn.github.com/images/modules/logos_page/GitHub-Mark.png" alt="Github logo" width="200"/>

Readme.md is fine format, but we wanted to have blog like docs
about our automation tasks too. But how to keep them in one place?
And whatever comes further along.
So why not take [Github Pages](https://github.io) into use?
There you get familiar git way to contribute to articles we share.

## Having nicer look and feel

<img src="https://getbootstrap.com/docs/4.1/assets/img/bootstrap-stack.png" alt="Bootstrap logo" width="200"/>

Then we need some way to display the articles nicely, hopefully
in blog format. This is now the first try with
[Bootstrap blog](https://getbootstrap.com/docs/4.1/examples/blog/)
framework. [See my first pretty try-out here](https://redhatnordicssa.github.io/index-bs.html)!

## Blog engine

That turned out to be nice layout, let's see if we can use it with
[Jekyll](https://jekyllrb.com/) blog framework. This page is now the first
result of it. To get started, e.g. watch
[these great video tutorials](https://jekyllrb.com/tutorials/video-walkthroughs/)

![Jekyll Logo](https://jekyllrb.com/img/logo-2x.png)

By applying those above, I have now created our blog. Welcome. Next thing
to do is to apply Bootstrap to the blog,
[here's tutorial](https://experimentingwithcode.com/creating-a-jekyll-blog-with-bootstrap-4-and-sass-part-1/).
Who will do the first PR for it?

Here we are. How about it? Does it seem reasonable? It's a
start at least! Let's see where it goes. Comments on twitter please.

Just PR for the submit button, feel free to start by either blogging or applying the bootstrap :D

So get git cloning: ```git clone git@github.com:RedHatNordicsSA/RedHatNordicsSA.github.io.git```


Yours,
 [@ikkeT](https://twitter.com/ikkeT)

## Update on Dec 21 2018

I switched to [hacker theme](https://github.com/pages-themes/hacker) just because I can! No seriously, I didn't want drop to scss level, I rather have pre-made theme. And boy had I have to try many of them until found one with working ruby dependencies.
