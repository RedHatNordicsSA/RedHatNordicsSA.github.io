---
layout: post
title:  "Shell scripting using Clojure and Babashka"
date:   2020-10-26 19:20:00 +0200
tags: clojure babashka shellscripting
author: tfriman
---

<p><banner_h>Babashka and FZF: match made in heaven!</banner_h></p>
Red Hat released a while ago [OpenShift Update
Service](https://www.openshift.com/blog/openshift-update-service-update-manager-for-your-cluster)
which has a handy [REST
API](https://api.openshift.com/?urls.primaryName=Upgrades%20information%20service#/default/getGraph)
for getting JSON describing available versions and upgrade
paths. There is an UI for that as well but a colleague of mine pasted
this on chat:

```
export CURRENT_VERSION=4.4.26;
export CHANNEL_NAME=stable-4.5;

curl -sH 'Accept:application/json' \
"https://api.openshift.com/api/upgrades_info/v1/graph?channel=${CHANNEL_NAME}" | \
jq -r --arg CURRENT_VERSION "${CURRENT_VERSION}" '. as $graph |
$graph.nodes |
map(.version=='\"$CURRENT_VERSION\"') |
index(true) as $orig |
$graph.edges |
map(select(.[0] == $orig)[1]) |
map($graph.nodes[.].version) |
sort_by(.)'
```
And here is the result of above command at the time of writing:

```
[
  "4.4.27",
  "4.5.13",
  "4.5.14",
  "4.5.15"
]
```

While I appreciated this jq beauty a horrible FOMO hit me. I felt
agony for not knowing how to utilize
[jq](https://stedolan.github.io/jq/manual/) for JSON filtering and
manipulation. I also saw [an another jq
script](https://github.com/openshift/cincinnati/blob/master/hack/graph.sh)
related to this and the decision was clear: I chose not to educate
myself with jq specific filters but to utilize Clojure for
manipulating the JSON for fun and profit. After all, the jq filtering
kind of looked like a functional pipeline and thus it should be an
easy task to reproduce the functionality. In addition I've been
wanting to use [Babashka](https://github.com/borkdude/babashka) for
something and this opportunity was too good to pass.

To follow the path chronologically, first I had to fathom the format
of the JSON file. I retrieved an example response (and formatted it
using jq, but that "filter" I knew how to use!):

```
curl -sH 'Accept:application/json' "https://api.openshift.com/api/upgrades_info/v1/graph?channel=stable-4.5" | jq .
```

An excerpt of that file containing only two versions and an upgrade path between them:
```
{
  "nodes": [
    {
      "version": "4.5.6",
      "payload": "quay.io/openshift-release-dev/ocp-release@sha256:0147ab7622969c1cde71e8e5eb8796e8245137fdbf3a5cae63017189a9060f86",
      "metadata": {
        "description": "",
        "io.openshift.upgrades.graph.previous.remove_regex": "4\\.4\\.1[12]",
        "io.openshift.upgrades.graph.release.channels": "candidate-4.5,fast-4.5,stable-4.5,candidate-4.6",
        "io.openshift.upgrades.graph.release.manifestref": "sha256:0147ab7622969c1cde71e8e5eb8796e8245137fdbf3a5cae63017189a9060f86",
        "url": "https://access.redhat.com/errata/RHBA-2020:3330"
      }
    },
    {
      "version": "4.5.3",
      "payload": "quay.io/openshift-release-dev/ocp-release@sha256:eab93b4591699a5a4ff50ad3517892653f04fb840127895bb3609b3cc68f98f3",
      "metadata": {
        "description": "",
        "io.openshift.upgrades.graph.previous.remove_regex": ".*",
        "io.openshift.upgrades.graph.release.channels": "candidate-4.5,fast-4.5,stable-4.5",
        "io.openshift.upgrades.graph.release.manifestref": "sha256:eab93b4591699a5a4ff50ad3517892653f04fb840127895bb3609b3cc68f98f3",
        "url": "https://access.redhat.com/errata/RHBA-2020:2956"
      }
    }
    ],
  "edges": [
    [
      1,
      0
    ]
    ]
}

```

So field "nodes" has all the versions and according their index there
are "edges" denoting possible upgrade paths. In this example index 1
(v4.5.3) has upgrade path to index 0 (v4.5.6). Armed with this info I
was able to hack this:

```
#! /usr/bin/env bb
;; Fetches OpenShift upgrade paths
;; usage: ./ocp-available-updates.sh # asks for channel and version
;;  ./ocp-available-updates.sh channel # asks only for version
;;  ./ocp-available-updates.sh channel version # fixed stuff, no fzf

(require '[clojure.java.io :as io]
         '[clojure.java.shell :refer [sh]]
         '[cheshire.core :as json]
         )

(import 'java.lang.ProcessBuilder$Redirect)

(def channels ["stable-4.5" "stable-4.4" "stable-4.3"
               "fast-4.5" "fast-4.4" "fast-4.3"])

(defmacro with-filter
  [command & forms]
  `(let [sh#  (or (System/getenv "SHELL") "sh")
         pb#  (doto (ProcessBuilder. [sh# "-c" ~command])
                (.redirectError
                 (ProcessBuilder$Redirect/to (io/file "/dev/tty"))))
         p#   (.start pb#)
         in#  (io/reader (.getInputStream p#))
         out# (io/writer (.getOutputStream p#))]
     (binding [*out* out#]
       (try ~@forms (.close out#) (catch Exception e#)))
     (take-while identity (repeatedly #(.readLine in#)))))

(let [[channel version] *command-line-args*
      channel (or channel (first (with-filter "fzf"
                                   (dorun (map #(println %) channels)))))
      apiresponse (-> (sh "curl" "-H" "Accept:application/json"
                          (str "https://api.openshift.com/api/upgrades_info/v1/graph?channel=" channel))
                      :out
                      (json/parse-string true))
      version (or version (first (with-filter "fzf"
                                   (dorun (map println (map :version (:nodes apiresponse)))))))
      updates (as-> apiresponse $
                ((fn [m] (assoc m :versions (map :version (:nodes m)))
                   ) $)
                ((fn [m] (assoc m :vmap (map
                                         (fn [[a b]]
                                           [(nth (:versions m) a) (nth (:versions m) b)])
                                         (:edges m)))
                   ) $)
                (sort (map second (filter (fn [[a _]] (= version a)) (:vmap $))))
                )
      ]
  (println "Channel " channel)
  (println "Version:" version)
  (println "Available updates:" updates)
  )

```

As you can see, it is not as succinct as the jq version but on the
other hand, it does a bit more! It utilizes an excellent fuzzy
matching tool called [fzf](https://github.com/junegunn/fzf) for
capturing missing inputs (channel and/or version). I found [a blog
posting](https://junegunn.kr/2016/02/using-fzf-in-your-program/)
describing how to use fzf with Clojure. I also chose not to create
helper functions to make it more look like the jq version.

Complete source code is
[here](https://github.com/tfriman/ocp-available-updates-fetcher). I
also
[containerised](https://quay.io/repository/tfriman/ocp-available-updates-fetcher?tab=info)
this, usage example:

```
docker run -it --rm quay.io/tfriman/ocp-available-updates-fetcher:v1
```

To reproduce the original jq example:

```
docker run -it --rm quay.io/tfriman/ocp-available-updates-fetcher:v1 stable-4.5 4.4.26
Channel  stable-4.5
Version: 4.4.26
Available updates: (4.4.27 4.5.13 4.5.14 4.5.15)
```

If you just want to see this in action, view this:

[![asciicast](https://asciinema.org/a/zKQ2rOyDzJVBVr3UEZu3Lr93v.svg)](https://asciinema.org/a/zKQ2rOyDzJVBVr3UEZu3Lr93v?autoplay=1&rows=30)


# Conclusions

Overall I was very happy with Babashka, it starts blazingly fast due
to the native compilation and has nice error messages (for some reason
I got used to those during the hacking phase;-) This simple exercise
proved me Clojure can be used for shell scripting as well :)

Thanks for Borkdude a.k.a. Michiel Borkent and Junegunn Choi for
creating and open sourcing these excellent tools!

# Edit 2020-10-27

I got a question on LinkedIn about overall experience and difference
to bash scripting (Thanks Antti Rintala). Experience was a pleasant
one and I got this done pretty quickly. I wrote the script in the old
way how I'm used to develop shell scripts: write script using Emacs
and then run it on terminal, rinse and repeat until done. I should
have used REPL for the development and get things done even
faster. Compared to ordinary shell scripts Clojure (or basically any
'proper' programming language suitable for shell scripting like
Python) provides a much better flow control. I also could have easily
added test to verify the functionality.

On the other hand, ordinary shell scripting has a great advantage: it
is very easy to pipe data between tools even interactively. To employ
fzf on this Babashka script I had to copy&paste macro for in/out
redirecting.
