---
layout: post
title:  "Use Ansible and Let's Encrypt to get (OpenShift) proper SSL certs"
date:   2019-03-10 15:20:00 +0200
tags:
 - openshift
 - ansible
author: ikke
---

<p><banner_h>The Challenge</banner_h></p>  
For those often rebuilding OpenShift
environments, it is ugly to have invalid SSL certificates for the site. Or any
web site, for that matter. It requires you to set all tools and browsers to
ignore invalid SSL certs. It might even break your testing. You test your
OpenShift upgrades and changes in separate test cluster instead of production,
right ;) ?

![letsencrypt-logo](assets/images/le-logo-lockonly.png){:height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![ansible-logo](assets/images/ansible_logo.png){:height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![openshift-logo](assets/images/openshift-logo.png){:height="80px"}

I describe here how I use Ansible, Let's Encrypt and Route53 DNS to set up my
test/demo OpenShift cluster with proper certificates. The method would apply to
pretty much any web service you have, not just OpenShift. Check out
[Let's Encrypt website](https://letsencrypt.org/) if you are not familiar with
the wonderful free certificate service.

<p><banner_h>The Environment</banner_h></p>

This write-up is about test environments, so I start by describing mine. I have
rented a server which comes with CentOS. I use Ansible to install all OpenShift
and virtualization requirements into it. Then I create 5 VMs on top of it,
and install OCP in there. I also install [Cockpit](https://cockpit-project.org/)
for having graphical panel to control and get visibility into environment. You
probably have proper private cloud or virtual environment for this, I don't.

![picture of architecture](assets/images/ocp-certs-pic.png)

BTW, if you find such setup interesting, all [ansibles are in git](https://github.com/ikke-t/ocp-libvirt-infra-ansibles) for you to
re-use.

<p><banner_h>Use cases</banner_h></p>

Here are some use cases "for backlog"

1. As an admin, I want proper FQDN for my cluster
2. As an  admin, I want properly secured Cockpit web GUI on host
3. As an OCP user, I want properly secured OpenShift web GUI
4. As an OCP user, I want properly secured OpenShift APIs
5. As an application user, I want web services on top of OpenShift properly
   secured

In short, the cluster looks ugly with confirmations about invalid certs, and
tools and automation on top of OCP likely breaks without valid certs.

<p><banner_h>Pre-Requisites</banner_h></p>

Get domain. This is the hardest part. You need to think of cool name here :) I
ended using my "container school" domain, konttikoulu.fi. To get domain, follow
[AWS
instructions](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/registrar.html),
out of scope of this blog.

You need a way to prove you own the domain you request the certificate for. Like
mentioned, I use Route53 from AWS to control my domain name. It's nice that
Let's Encrypt -service's certbot -tool supports it out of box. There are variety
of other DNS provides supported by Let's Encrypt tooling, see e.g.
[certbot](https://certbot.eff.org/lets-encrypt/centosrhel7-other) and
[acme.sh](https://github.com/Neilpang/acme.sh) documentation.

There are at least two different ways of Let's Encrypt verifying your domain:

Let's encrypt's server talks to either:
1. Your DNS vendor to verify you own domain
2. Your web site (OpenShift router) and verifies that FQDN leads to your web
   page (see linked blog at bottom for this method)

In this case I use the DNS method. Nice part with it is that it works even if
your cluster would not be reachable from internet. For this to work, I
followed [the instructions to create keys for certbot to use my
Route53](https://certbot-dns-route53.readthedocs.io/en/stable/) for
verification.

<p><banner_h>Changes required to install config</banner_h></p>

Let's get to business. Let's look at what needs to be done to have proper certs:

1. Install let's encrypt tools to bastion
1. Get certificates for
  1. Cockpit on host
  1. OpenShift API (same as web GUI)
  1. OpenShift Router cert (* -cert)
2. Modify OpenShift installer inventory to use FQDN
2. Modify OpenShift installer inventory to use certs
5. run the installer

I have my OpenShift install automation in git. I did this all in one commit
which you can [see in git
diff](https://github.com/ikke-t/ocp-libvirt-infra-ansibles/commit/162d765c03dcfcf4ef9c29cf3484228a6ee2b29b).
Well, actually two, as I got sloppy in first commit, I fixed some variable names
[in second
commit](https://github.com/ikke-t/ocp-libvirt-infra-ansibles/commit/ad82fc124fa29b021f6fc49d4a8e2e15cfa40d5a.)

<p><banner_h>Let's go through the changes step by step</banner_h></p>


## Setup epel repo for bastion to get certbot tool

Note that I also disable it by default, so that I don't accidentally install
non-supported versions of software, which might break installation.

```{%raw%}
- https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
package: name={{ pkgs }} state=present

- name: disable epel by default installations
lineinfile:
path: /etc/yum.repos.d/epel.repo
regexp: '^enabled=1'
line: 'enabled=0'

- name: install certbot from epel
yum:
name:
- certbot
- python2-certbot-dns-route53
enablerepo: epel
state: present{%endraw%}
```

## Check if we have previously acquired certificate

We setup cron later to keep cert up to date. So we don't want to unnecessarily
run setup again.

```{%raw%}
- name: Check if let's encrypt is already done on previous runs
  stat:
    path: "/etc/letsencrypt/live/{{ api_fqdn }}/fullchain.pem"
  register: cert_exists{%endraw%}
```

## Setup AWS key and acquire the cert

I drop the certbot's AWS access key into default AWS config file.
Note not to use your admin key here just for good practice!

```{%raw%}
- name: Lets encrypt cert for ocp
  block:

    - name: create awx secrets
      file:
        path: /root/.aws
        state: directory
        mode: 0600
    - copy:
        dest: /root/.aws/config
        mode: 0600
        content: |
          [default]
          aws_access_key_id={{aws_access_key_id}}
          aws_secret_access_key={{aws_secret_access_key}}{%endraw%}
```

## Acquiring the certs is one command

See the keys get created into ```/etc/letsencrypt``` directory. Notice to put
all FQDNs into cert via ```-d``` switch.

```{%raw%}
    - name: get let's encrypt cert
      command: >
        certbot certonly --dns-route53 --non-interactive --agree-tos
        --server https://acme-v02.api.letsencrypt.org/directory
        -m "{{ letsencrypt_email }}"
        -d "{{ api_fqdn }}" -d "{{apps_fqdn}}" -d "*.{{apps_fqdn}}"
      args:
        creates: "/etc/letsencrypt/live/{{ api_fqdn }}/fullchain.pem"{%endraw%}
```

## Setup periodic renewal of certs

Let's encrypt issued certs live only for 90 days. So if I keep using the same
bastion, I will always have valid certs in the same place, updated by certbot in
cron. Remember there is [another ansible you need to run to put renewed certs
into OpenShift
cluster](https://docs.openshift.com/container-platform/3.11/install_config/redeploying_certificates.html)
if you keep running this env for long.

```{%raw%}
    - name: put let's encrypt to renew certs periodicly
      cron:
        name: "renew certbot certs"
        minute: "20"
        hour: "02"
        weekday: "2"
        job: "certbot renew &> /dev/null"{%endraw%}
```

## Keep it optional

This is for myself only. I wanted to keep an option to use the same ansibles
without using certbot. If I don't define AWS keys, it won't do certs. This is
work in progress still... I forgot to make it conditional in inventory template.

```{%raw%}
  when:
    # we do let's encrypt if AWS keys are provided, and we do it only once
    - aws_access_key_id is defined
    - aws_secret_access_key is defined
    - cert_exists.stat.islnk is not defined{%endraw%}
```

## Ensure installer is allowed to access keys and certs

By default keys are protected. This allows installer access to keys.

```{%raw%}
- name: Allow installer access to ssl certs
  file:
    path: "{{ item }}"
    state: directory
    mode: 0705
  with_items:
    - /etc/letsencrypt/live
    - /etc/letsencrypt/archive

- name: Allow installer access to ssl key
  file:
    path: /etc/letsencrypt/archive/{{ api_fqdn }}/privkey1.pem
    state: file
    mode: 0704{%endraw%}
```

## Set installer inventory to use the FQDN and certs

You need to tell installer to set FQDN to be used to your cluster. Below I set
it for master API and some services. See also the definition for certs.

```{%raw%}
openshift_master_cluster_public_hostname={{ api_fqdn }}
openshift_master_default_subdomain={{apps_fqdn}}
openshift_logging_kibana_hostname=logging.{{apps_fqdn}}

openshift_master_named_certificates=[{"names": ["{{ api_fqdn }}"], "certfile": "/etc/letsencrypt/live/{{ api_fqdn }}/fullchain.pem", "keyfile": "/etc/letsencrypt/live/{{ api_fqdn }}/privkey.pem", "cafile": "/etc/letsencrypt/live/{{ api_fqdn }}/chain.pem"}]

openshift_hosted_router_certificate={"certfile": "/etc/letsencrypt/live/{{ api_fqdn }}/fullchain.pem", "keyfile": "/etc/letsencrypt/live/{{ api_fqdn }}/privkey.pem", "cafile": "/etc/letsencrypt/live/{{ api_fqdn }}/chain.pem"}

openshift_hosted_registry_routecertificates={"certfile": "/etc/letsencrypt/live/{{ api_fqdn }}/fullchain.pem", "keyfile": "/etc/letsencrypt/live/{{ api_fqdn }}/privkey.pem", "cafile": "/etc/letsencrypt/live/{{ api_fqdn }}/chain.pem"}{%endraw%}
```

# Host's Cockpit Certificates

![cockpit-logo](assets/images/cockpit.png){: height="80px" align="center"}

There is similar logic for host server's Cockpit. [See the commit in git](https://github.com/ikke-t/ocp-libvirt-infra-ansibles/commit/a3bbddc153d761066b73924d9d340de1a4500f98),
I don't copy paste it here as it's almost identical to above steps.

You could do the same for Cockpit at OpenShift master. I didn't bother, as I use
the Cockpit from host, and tell that to connect to OpenShift master's Cockpit.
So my browser never hit's it directly. Did you know you can have several host's
Cockpits used from one Cockpit web GUI?

Note, if you copy paste the code in previous link, there is an erroneous
redirect. Change ```>>``` to ```>``` in there, to avoid stacking the certs into one
file. It's fixed in later commits.

<p><banner_h>That's it, folks, we are done!</banner_h></p>

So now, with those few ansible lines, and running installer, you could turn your
OpenShift and Cockpit to use proper certs. I believe they are pretty re-useable
to any SSL service you might have. Happy safe internetting for you and your
OpenShift users!


# Links for related docs and blogs

Here are links to official documentation, and another blog that has a bit
different way of setting the router address. It's definitely better for router.
It is based on the non DNS way of ensuring web site, and will generate
application specific certs instead of wildcard cert. It's a good reference
for the next step for you to take.

* [OpenShift SSL certs documentation](https://docs.openshift.com/container-platform/3.11/install_config/certificate_customization.html)
* [OpenShift SSL certs renewing](https://docs.openshift.com/container-platform/3.11/install_config/redeploying_certificates.html)
* [Great blog from Red Hat colleagues](https://eti.io/dynamic-ssl-certificates-using-letsencrypt-on-openshift/)
  to do certs the other way for apps
* [Certbot tool for Let's Encrypt certicates](https://certbot.eff.org/lets-encrypt/centosrhel7-other)
* [acme controller for OpenShift](https://github.com/tnozicka/openshift-acme)
* [acme script for Let's Encrypt certificates](https://github.com/Neilpang/acme.sh) supports variety of DNS
  services.

BR,  
ikke

PS, Let's Encrypt [takes donations here](https://letsencrypt.org/donate/).
