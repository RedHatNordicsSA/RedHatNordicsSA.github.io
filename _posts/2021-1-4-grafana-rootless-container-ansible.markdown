---
layout: post
title:  "Automate rootless containers with Ansible - Grafana"
date:   2021-1-4 12:00:00 +0200
tags:
  - containers
  - ansible
  - grafana
author: ikke
---

<p><banner_h>The Challenge: Ansible automate grafana in rootless container</banner_h></p>
I have rather much services and home automation running at home. I want to monitor
their state and get alerted if something is out of the normal. I had grafana set up
for the job, but now had to move servers. While at it, I want it fully automatized
and running in rootless container. How can I automate rootless containers?

The business reason for spending time for this is to show how one could
automate Edge computing environments in Edge/IoT small scale devices. There
you'd develop and test your solutions in Kubernetes, and push the software as
containers to small edge devices using Ansible Tower and this kinda setup for
example.

![grafana-logo](assets/images/grafana_logo.svg){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![systemd-logo](assets/images/systemd-logo.png){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![podman-logo](https://github.com/containers/libpod/raw/master/logo/
podman-logo.png){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![ansible-logo](assets/images/ansible_logo.png){:height="80px"}

Oh, and why rootless? Well, it's yet another layer of security. Even if the
evil blackhat managed to fight breaking into your container, managed to find
a security hole to punch through container and SELinux, there is yet another
layer of trouble. At this point one is not root, but just a regular user,
which has no privileges in the system.

Also, you don't necessarily need root privileges at all in target to run
containers.

# Description of the environment

I've been writing earlier how to ansible podman containers, but only now added the
rootless option to my ansible role. Also, for my delight, I noticed there is [ansible
module for grafana](https://github.com/ansible-collections/community.grafana) setup.

In this article I'll setup grafana with a dummy dashboard and some test users.
Normally I connect grafana to my influxdb data source to where my
[OPNSense](http://opnsense.org/) firewall/router pushes HAproxy load balancer
stats, along with tons of other stuff like [OpenHAB](https://www.openhab.org/)
home automation and monitor statuses. Grafana then serves me with alerts when
necessary.

I describe what does it take to make Ansible create persistent containers using
podman using user only privileges for container. Compared to Docker, it's
systemd which manages these user processes here. I'll focus on the changes
needed around systemd.

![setup](assets/images/podman-user.png)

# Systemd changes needed in ansible

I use the same
[podman_container_systemd](https://galaxy.ansible.com/ikke_t/podman_container_systemd)
role as I used in my earlier
[ansible podman systemd articles](https://redhatnordicssa.github.io/ansible-podman-containers-1).
Let's go through here what had to be changed to make it work in user mode, as
well changed needed in Fedora CoreOS and deriative servers intended to run only
containers. I use RHEL 8 Edge during the process.

## User systemd services

First of all the service files need to be put into users home dir instead of
system config dirs ```/etc/systemd/system``` or ```/usr/lib/systemd/system```.
Technicly, there is also a user dir, but I want them private to user.

[Here's how](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/tasks/main.yml#L12)
```{% raw %}
    - name: set systemd dir if user is not root
      set_fact:
        service_files_dir: "{{ user_info.home }}/.config/systemd/user"
        systemd_scope: user
      changed_when: false

    - name: ensure systemd files directory exists if user not root
      file:
        path: "{{ service_files_dir }}"
        state: directory
        owner: "{{ container_run_as_user }}"
        group: "{{ container_run_as_group }}"
{% endraw  %}```

## Persistent DBUS session

The biggest hurdle for me was to understand how such service user gets
DBUS session which remains there over boots, even if user never logs in.
This happens by setting up a "lingering" session for user, which activates
DBUS for given user at boot.

[See here](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/tasks/main.yml#L159)


```{% raw %}
  - name: Check if user is lingering
    stat:
      path: "/var/lib/systemd/linger/{{ container_run_as_user }}"
    register: user_lingering
    when: container_run_as_user != "root"

  - name: Enable lingering is needed
    command: "loginctl enable-linger {{ container_run_as_user }}"
    when:
      - container_run_as_user != "root"
      - not user_lingering.stat.exists
{% endraw  %}```

Next thing is that systemd commands need to be all
[executed in user scope](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/tasks/main.yml#L33).
As we don't log in as target user, we achieve this by setting environment variable
for ```xdg_runtime_dir``` to find user lingering DBUS session later.

[See here](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/tasks/main.yml#L33)

```{% raw %}
- name: set systemd runtime dir
  set_fact:
    xdg_runtime_dir: "/run/user/{{ container_run_as_uid.stdout }}"
  changed_when: false

- name: set systemd scope to system if needed
  set_fact:
    systemd_scope: system
    service_files_dir: '/etc/systemd/system'
    xdg_runtime_dir: "/run/user/{{ container_run_as_uid.stdout }}"
  when: container_run_as_user == "root"
  changed_when: false
{% endraw  %}```

## Systemd service file target change

Systemd session file has to be changed to default target when run in rootless.
[See here](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/templates/systemd-service-single.j2#L26)
```{% raw  %}
[Install]
{% if container_run_as_user == 'root' %}
WantedBy=multi-user.target
{% endif %}
{% if container_run_as_user != 'root' %}
WantedBy=default.target
{% endif %}
{% endraw  %}```

## User systemd commands

Now that all required variables are set, we tell ansible to use
the DBUS session
[like this](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/handlers/main.yml#L12)

```{% raw %}
- name: start service
  become: true
  become_user: "{{ container_run_as_user }}"
  environment:
    XDG_RUNTIME_DIR: "{{ xdg_runtime_dir }}"
  systemd:
    name: "{{ service_name }}"
    scope: "{{ systemd_scope }}"
    state: started
{% endraw  %}```

Note that we switch to given user, and set the runtime dir to catch the dbus.
Also the scope is set to user instead of system.

## RPM-OSTREE package handling

As I said, I want to run my containers in minimal Fedora-CoreOS based
machines. There is a quirk that ansible doesn't have proper package
module for such. So checking and installing packages needs to be worked
around
[like this](https://github.com/ikke-t/podman-container-systemd/blob/9a0bf79a47b569050e09dcfb3a367dffbf39b4d8/tasks/main.yml#L227)

```{% raw %}
- name: ensure firewalld is installed (on fedora-iot)
    tags: firewall
    command: >-
      rpm-ostree install --idempotent --unchanged-exit-77
      --allow-inactive firewalld
    register: ostree
    failed_when: not ( ostree.rc == 77 or ostree.rc == 0 )
    changed_when: ostree.rc != 77
    when: ansible_pkg_mgr == "atomic_container"

  - name: reboot if new stuff was installed
    reboot:
      reboot_timeout: 300
    when:
      - ansible_pkg_mgr == "atomic_container"
      - ostree.rc != 77
{% endraw  %}```

You probably don't want to install anything there, but just in case you do,
this handles the required reboot depending if anything got installed.

At this point we are pretty much done with the changes.

# Want to try this out?

In case you want to try this yourself, and why wouldn't you :D, 
this chapter gives you the commands to set this to your own environment. I use
RHEL8 on my laptop, and RHEL Edge as the target VM. I also tried this with
RHEL8 and Fedora-IoT targets.

I like to keep everything related to task in one directory, including required
collections and roles. So let's setup such directory:

```{% raw %}
sudo dnf install ansible
git clone https://github.com/ikke-t/ansible-podman-sample.git
cd ansible-podman-sample
{% endraw  %}```

and install the dependencies:

```{% raw %}
ansible-galaxy collection install -r collections/requirements.yml -p collections
ansible-galaxy role install -r roles/requirements.yml -p roles
{% endraw  %}```

And run the playbook.
```{% raw %}
ln -s roles/ikke_t.grafana_podman/tests/test.yml run-container-grafana-podman.yml
ansible-playbook -i edge, -u cloud-user -b \
  -e container_state=running \
  -e ansible_pkg_mgr=atomic_container \
  run-container-grafana-podman.yml
{% endraw  %}```

Where the following needs to be changed to be applicable to your system:

* ```ansible_pkg_mgr``` is
  [needed only if target is RHEL Edge](https://github.com/ansible/ansible/issues/73084),
  otherwise remove the line.
* ```edge``` is my VM server ssh address.
* cloud-user is the sudo privileged user in target VM
 
And, drumroll, TADAA we have our grafana runnning in user session as container
(http://your_vm:3000):

![grafana dashboard](assets/images/grafana-dashboard.png)

Test users pushed over API using ansible grafana collection:

![grafana users](assets/images/grafana-users.png)

And how about my home setup? It's running on Fedora-IoT, and I don't need
backups, the whole setup can be rebuilt in few minutes by ansible.

Here it tracks HAproxy frontend for all my services:

![grafana haproxy](assets/images/grafana-haproxy.png)

And here I get alerts if house heating has stopped due fault:

![grafana temperatures](assets/images/grafana-temps.png)


## To debug things locally in target

As we run containers as user, and this ansible role places them under user
context of systemd, you need to setup DBUS related env variable to get to use
systemd as given user. However, the user does not have login credentials. You
need to switch manually to user using su, sudo will not work as original UID
would show your ssh user UID.

```{% raw %}
su - root
su - grafana
export XDG_RUNTIME_DIR=/run/user/$UID
{% endraw  %}```

After setting ```XDG_RUNTIME_DIR``` you get to use ```systemd --user``` command to
investigate your systemd service set for the container:

```{% raw %}
[cloud-user@edge ~]$ su - 
Password: 
Last login: Thu Dec 31 13:03:43 EET 2020 on pts/2
[root@edge ~]# su - grafana
Last login: Thu Dec 31 13:04:41 EET 2020 on pts/1
(failed reverse-i-search)`export': ^C
[grafana@edge ~]$ export XDG_RUNTIME_DIR=/run/user/$UID
[grafana@edge ~]$ 
[grafana@edge ~]$ systemctl --user status grafana-container-pod-grafana.service 
● grafana-container-pod-grafana.service - grafana Podman Container
   Loaded: loaded (/var/home/grafana/.config/systemd/user/grafana-container-pod-grafana.service; enabled; ve>
   Active: active (running) since Thu 2020-12-31 13:06:35 EET; 29min ago
  Process: 1122 ExecStartPre=/usr/bin/rm -f /tmp/grafana-container-pod-grafana.service-pid /tmp/grafana-cont>
 Main PID: 1126 (podman)
   CGroup: /user.slice/user-1002.slice/user@1002.service/grafana-container-pod-grafana.service
           ├─1126 /usr/bin/podman run --name grafana --rm -p 3000:3000/tcp -e GF_INSTALL_PLUGINS=flant-statu>
           ├─1158 /usr/bin/podman run --name grafana --rm -p 3000:3000/tcp -e GF_INSTALL_PLUGINS=flant-statu>
           ├─1167 /usr/bin/podman
           ├─1200 /usr/bin/slirp4netns --disable-host-loopback --mtu 65520 --enable-sandbox --enable-seccomp>
           ├─1206 /usr/bin/fuse-overlayfs -o lowerdir=/var/home/grafana/.local/share/containers/storage/over>
           ├─1214 containers-rootlessport
           ├─1227 containers-rootlessport-child
           ├─1240 /usr/bin/conmon --api-version 1 -c 2675941bf4743ff26860ff2e84ceaaae78f2fcfbe3fef218e0cfee9>
           └─2675941bf4743ff26860ff2e84ceaaae78f2fcfbe3fef218e0cfee914aa96b37
             └─1251 grafana-server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packagi>
[grafana@edge ~]$ podman ps
CONTAINER ID  IMAGE                             COMMAND  CREATED         STATUS             PORTS                   NAMES
2675941bf474  docker.io/grafana/grafana:latest           29 minutes ago  Up 29 minutes ago  0.0.0.0:3000->3000/tcp  grafana
{% endraw  %}```

# Tearing down the setup, nuke it all!

Now that we saw how it works, it's time to clean up stuff. Beware that the
"nuke=true" option removes both the grafana user and the data, so make sure
there is nothing important you could loose. Again, remove the pkg_mgr if not
on RHEL Edge target.

```{% raw %}
ansible-playbook -i edge, -u cloud-user -b \
  -e container_state=absent \
  -e ansible_pkg_mgr=atomic_container \
  -e nuke=true \
  run-container-grafana-podman.yml
{% endraw  %}```

# Conclusion

Podman and systemd are good mechanism to run containers in small setups where
kubernetes would be overkill. Ansible is robust way to create such setups.
Sometimes, like here, you don't even need backups, as target is super easy
to create from scratch by ansible.

For further reading about podman and systemd, please see:
* [podman-generate-systemd man page](http://docs.podman.io/en/latest/markdown/podman-generate-systemd.1.html#:~:text=DESCRIPTION,units%20on%20the%20remote%20system.)
* [Running containers with Podman and shareable systemd services](https://www.redhat.com/sysadmin/podman-shareable-systemd-services)
* [Improved systemd integration with Podman 2.0](https://www.redhat.com/sysadmin/improved-systemd-podman)
