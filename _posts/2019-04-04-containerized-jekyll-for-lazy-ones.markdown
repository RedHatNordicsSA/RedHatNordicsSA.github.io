---
layout: post
title:  "Using containerized Jekyll"
date:   2019-04-06 15:20:00 +0200
tags: jekyll
author: tfriman
---

<p><banner_h>Running Jekyll in container to create an entry to this blog</banner_h></p>
How to run Jekyll in a container and not polluting your gear with gems.

This is my first blog posting ever and I was planning to write about
something completely different than this so bear with me. I'm running
MacOS and was pretty disappointed when I tried running ```brew install
jekyll``` and then found out that there is no available formulae with
the name "jekyll". After that I checked Peter's [blog
posting](./install-jekyll-on-rhel) and was pondering if I could get
away without installing any ruby gems to my laptop. It turned out to
be quite straightforward with containers.

First you need get a container runtime. That is left as an exercise to
the reader. I'm running these on MacOS 10.14.4 using Docker, there are
other alternatives too like [podman](https://podman.io) for Linux
which is cli-compatible to docker cli.

Clone the blog source:

```
git clone git@github.com:RedHatNordicsSA/RedHatNordicsSA.github.io.git
```

Move to the freshly cloned dir.

```
cd RedHatNordicsSA.github.io
```

Start containerized Jekyll. This will take some time due to fetching gems.

```
docker run --rm \
  --volume="$PWD:/srv/jekyll" \
  -it \
  -p 4000:4000 \
  jekyll/jekyll \
  jekyll serve
```

After that you can commit the running container image with fetched gems.

```
docker ps
```

Find out the running Jekyll container and commit and tag it using id,
here id being "f988c4ba1873".

```
docker commit f988c4ba1873
ha256:942991114b0340e5f3ad2451bbec8e00516508f0940aea2c92753c6b70231d0d
docker tag 942991114b03 jekyll_nordicssa_blog
```

Now you can kill the previous running container and start a new one using tag.

```
docker run --rm \
  --volume="$PWD:/srv/jekyll" \
  -it \
  -p 4000:4000 \
  jekyll_nordicssa_blog \
  jekyll serve
```

I'm quite certain there are easier ways to accomplish this but I have
to concentrate on the original blog posting idea next. If you find out
issues with this post you can always create an issue and leave it to
the github. I won't guarantee any SLA for reviewing and fixing those
though!

Happy blogging!

References:

Ilkka's original [Writing new blog on Jekyll](./new-post)
