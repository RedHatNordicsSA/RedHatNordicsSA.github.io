---
layout: post
title:  "Clojure development using OpenShift, part 1"
date:   2019-05-08 09:20:00 +0200
tags: clojure openshift s2i
author: tfriman
---

<p><banner_h>How to get your Clojure apps running on OpenShift</banner_h></p>
[Clojure](https://www.clojure.org) is a fantastic language but how you
can run Clojure apps on top of OpenShift?

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

<b>Option 1</b>: Add source-to-image (S2I) builder to OpenShift and let the
OpenShift take your application's source code from the SCM and build
it and create a container out of the binary and then run it. This is
the way things will be done in this posting series.

<b>Option 2</b>: use created uberjar and let OpenShift do it's magic with
binary builds. This may be covered later on but not now.

<b>Option 3</b>: use created uberjar and create your container image outside OpenShift. Run that image on OpenShift. Not coverered in this post either.

I assume you know how S2I works but if not check out
[this](https://github.com/openshift/source-to-image). There are S2I
builders for many languages out of the box but none for
Clojure. [Here](https://github.com/tfriman/s2i-clojure) is a simple
one using [lein](https://leiningen.org). It contains instructions how
to install the builder to your cluster but for completeness sake I’ll replicate them here.

<p><banner_h>Custom S2I installation to OpenShift 3.11</banner_h></p>

Note that these instructions have been tested with OpenShift version
3.11.

Login to your OpenShift cluster

```
oc login
```

You can install this to any namespace but "openshift" is visible to all by default and it is used when service catalog searches for builders so let's use it.

```
oc new-build https://github.com/tfriman/s2i-clojure#v1.0.0 --name s2i-clojure -n openshift
```

You can follow the build

```
oc logs -f bc/s2i-clojure -n openshift
```

<p><banner_h>Usage example</banner_h></p>

Create a test project

```
oc new-project clj-test
```

After the build has finished you can test your new builder:

```
oc new-build s2i-clojure~https://github.com/tfriman/clj-rest-helloworld#v1.0.0 --name=clj-test
```

And follow the build

```
oc logs -f bc/clj-test
```

After the build has finished create a new app:

```
oc new-app clj-test
```

And open the service to the world:

```
oc expose svc/clj-test
```

See the url for the application

```{% raw %}
oc get routes clj-test --template='{{ .spec.host }}'
{% endraw %}```

And open it using your browser.

<p><banner_h>Making s2i-clojure builder visible to catalog</banner_h></p>

You can use your custom builder from command line without getting it
to catalog but it would be nice to get this to UI as well so here
goes:

First you need to edit the image stream:

```
oc edit is/s2i-clojure -n openshift -o json
```

Make it look like this, add the missing parts, don't remove things:

```
{
    "kind": "ImageStream",
    "apiVersion": "v1",
    "metadata": {
	"name": "s2i-clojure-builder",
	"annotations": {
	    "openshift.io/display-name": "S2I Clojure"
	}
    },
    "spec": {
	"tags": [
	    {
		"name": "latest",
		"annotations": {
		    "openshift.io/display-name": "S2I Clojure",
		    "description": "Build and deploy a Clojure app",
		    "iconClass": "icon-clojure",
		    "sampleRepo": "https://github.com/tfriman/clj-rest-helloworld",
		    "sampleRef": "v1.0.0",
		    "tags": "builder,clojure",
		    "version": "latest",
		    "supports": "clojure"
		},
		"from": {
		    "kind": "DockerImage",
		    "name": "s2i-clojure:latest"
		}
	    }
	]
    }
}

```

Wait for a while to catalog service to catch up and your Clojure S2I
builder should appear in the catalog under section "other".

Happy Clojure development using S2I!
