---
layout: post
title:  "Clojure development using OpenShift, part 1"
date:   2019-05-03 15:20:00 +0200
tags: clojure openshift s2i
author: tfriman
---

<p><banner_h>How to get your Clojure apps running on OpenShift</banner_h></p>
[Clojure](https://www.clojure.org) is a fantastic language but how you can run that on top of OpenShift?

[![clojure logo](https://www.clojure.org/images/clojure-logo-120b.png)](https://www.clojure.org)

First a disclaimer: I've never written a single line of Clojure that
has ended up in production nor I think I'm fluent with
Clojure. Instead I've been tinkering with Clojure for years now and
still find the moment when my Clojure program is running flawlessly
magical. It makes me feel very computer savvy when my lisp program is
working as expected. Your mileage hopefully varies and if you find
something that should be fixed with code or anything other please
leave an issue here at github, thank you! I'm also expecting that you
know your ways around Clojure and it's tooling, namely lein and same
applies with OpenShift and it's tooling. Also I'm aware that there are
other runtimes than JVM to run your Clojure applications but this
posting is about JVM and jars.

So you have your Clojure program ready to be deployed. How does one
get that running on OpenShift? Well, it's pretty straightforward but
there are multiple options how to make that happen:

Option 1: use created uberjar and let OpenShift do it's magic with
binary builds. This may be covered later on but not now.

Option 2: Add source-to-image (S2I) builder to OpenShift and let the
OpenShift take your application's source code from the SCM and build
it and create a container out of the binary and then run it. This is
the way things will be done in this posting series.

I assume you know how S2I works but if not check out
[this](https://github.com/openshift/source-to-image). There are S2I
builders for many languages out of the box but none for
Clojure. [Here](https://github.com/tfriman/s2i-clojure) is a simple
one using [lein](https://leiningen.org). It contains instructions how to install the builder to
your cluster, please follow them.

<p><banner_h>Making builds faster</banner_h></p>

You have multiple options how to make builds faster but they all rely
on the same method: re-using previous work in some form. One way is to
enable incremental S2I builds. Incremental S2I build means that
previously built image with dependencies (in this case dependency jar
files) is fetched and those dependencies stored on the previous run
are copied to new build.

In practice you can enable clojure-s2i build incremental flag like
this, using clj-test build from the s2i-clojure repo as an example:

```
oc patch bc/clj-test -p '{"spec":{"strategy": { "sourceStrategy": {"incremental": true } } } }'
```

Above patch command updates (creates, if it was missing) the key with
name 'incremental' under 'sourceStrategy' with value 'true'. You can
achieve the same using OpenShift web console and navigating to project
clj-test / builds / clj-test and selecting 'Actions / Edit YAML'

Example run when incremental is on:

```
# oc start-build clj-test --follow

build.build.openshift.io/clj-test-5 started
Cloning "https://bitbucket.org/tfriman/clj-rest-helloworld" ...
	Commit:	10f41ecc9b8469ba1c820ad115fe6f51b53911bf (removed .s2i)
	Author:	Timo Friman <tfriman@redhat.com>
	Date:	Mon May 6 16:24:06 2019 +0300
	Using docker-registry.default.svc:5000/openshift/s2i-clojure@sha256:d71a835bb1aaa9e5e5fbe9e5d7b6610f469ef588446798497764a335ccf4f2f1 as the s2i builder image
Pulling image "docker-registry.default.svc:5000/clj-test/clj-test:latest" ...
---> Restoring build artifacts...
---> Installing application source...
---> Building application from source...
Compiling demoapp.config
```
