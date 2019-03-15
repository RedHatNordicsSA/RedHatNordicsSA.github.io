---
layout: post
title:  "Install Jekyll on RHEL7"
date:   2018-11-19 12:30:00 +0200
tags: random
author: ikke
---

<p><banner_h>How I installed Jekyll on RHEL7</banner_h></p>
While I was creating this blog I needed to get [Jekyll](https://jekyllrb.com/)
tooling on my laptop. I happen to be fan of Linux, and decided to eat our own
dogfood. I recently installed RHEL7 on my laptop instead of the normal Fedora.

![Jekyll logo](https://jekyllrb.com/img/logo-2x.png)

Jekyll works fine on RHEL, but like often, you run into having stable (old) stuff
installed on your system. Jekyll will require more fresh ruby than what comes
by default in RHEL7. You need feature called
[Software Collections](https://developers.redhat.com/products/softwarecollections/overview/),
which is a way to provide developers more fresh set of tools, with shorter
lifecycle for support.

Here are the command you need to install Jekyll on RHEL7:

```
sudo subscription-manager repos --enable rhel-workstation-rhscl-7-rpms
sudo yum install rh-ruby23 rh-ruby23-rubygem-bundler rh-ruby23-ruby-devel zlib-devel
sudo yum group install "Development Tools"
scl enable rh-ruby23 bash
gem install jekyll
jekyll new redhatnordicssa
cd redhatnordicssa
bundle exec jekyll serve
```

## What is that gibberish all about?

The above commands are explained here line by line:

1. enable repository for Software collections (SC)
2. install newer ruby stuff from SC
3. enter SC shell
4. install jekyll using ruby gem
5. create new Jekyll site
6. enter the directory for your site
7. start local server so you can test your site on http://localhost:4000

Then you can go ahead and start following [Jekyll tutorials](https://jekyllrb.com/tutorials/video-walkthroughs/). Have fun!

BR, [ikke](https://twitter.com/ikkeT)
