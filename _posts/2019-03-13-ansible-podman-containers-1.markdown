---
layout: post
title:  "Automate Podman Containers with Ansible 1/2"
date:   2019-03-13 07:30:00 +0200
tags:
  - containers
  - ansible
author: ikke
---

<p><banner_h>The Challenge</banner_h></p>
Container tooling has improved a lot recently. Nowadays there is lot of progress
being done around OCI (Open Container Initiative) compatible tools.
[Podman](https://github.com/containers/libpod), CRI-O and Buildah are new tools
to build and run containers. I describe here how I changed my hobby projects'
containers from Docker into Podman using Ansible to automate them. Having
Ansible wrapping helps maintenance and rebuild.

![podman-logo](https://github.com/containers/libpod/raw/master/logo/podman-logo.png){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![ansible-logo](assets/images/ansible_logo.png){:height="80px"}


# Environment description

Like said, this is my hobby environment, but it's quite professional. This could
be IoT edge device case. These instructions would apply to any environment where
you run containers on host. My environment is not Kubernetes cluster, but single
host, so I use podman to run containers, and Systemd to control their state.
Previously I was still running them using Docker, but it's not required anymore.

I run services for home automation, and generic nerding at home:
* [OpenHAB](https://www.openhab.org/) for home automation
* [Node-Red](https://nodered.org/) for home automation
* [Grafana](https://grafana.com/) for metering things and getting alerts  
* [NextCloud](https://nextcloud.com/) For syncing files and many other personal
  cloud tools
* [Gogs](https://gogs.io/) lightweight git server for storing code and
  configurations
* [PhpBB](https://www.phpbb.com/) for hosting a forum
* [Unifi Controller](https://www.ui.com/software/) for managing my WLAN
* [Lighttpd](https://www.lighttpd.net/) web servers
* [WeeChat](https://weechat.org/) for IRC relay. Yes, you read right. I still
  do IRC. IRC is still cornerstones of open source development.
* [AWX](https://github.com/ansible/awx) Ansible Tower upstream for helping
  automating it all

# Components

I very much like Ansible. It helps me automate installation of stuff, and also
managing updates. And also removing stuff which is not needed any longer. In
this blog I use it to manage Systemd service definition files for containers.
As podman is daemonless, I use systemd to make sure containers come and go as
I wish across reboots.

[Here is podman_container_systemd Ansible
role](https://galaxy.ansible.com/ikke_t/podman_container_systemd) that I wrote
to help me maintain systemd definitions for podman. I wrote a role as there are
repeating actions all containers need. So now they are in one place. I also
wrote [little helper Ansible role
container_image_cleanup](https://galaxy.ansible.com/ikke_t/container_image_cleanup)
to clean up container images I no longer need. In my nerding I run out of disk
space due excessive amount of unused container images :)

# What I want to do for each container

I want for every container existence of:
* systemctl service file
* firewall ports open if needed
* update container images by rerun (software version update)
* shared volume if needed


First two are taken care by [podman_container_systemd Ansible
role](https://galaxy.ansible.com/ikke_t/podman_container_systemd). The rest I
do in playbook for each container.

# Import required roles

You'll need to pull the mentioned roles for the above playbooks to run, let's
put them into ```roles/``` directory:

```
mkdir roles
cat >>roles/requirements.yml<<EOF
---

- src: ikke_t.container_image_cleanup
  name: container_image_cleanup

- src: ikke_t.podman_container_systemd
  name: podman_container_systemd
EOF

ansible-galaxy --roles-path roles install -r roles/requirements.yml
```

# Create playbooks

Here is now all it takes to run such containers, this is OpenHAB
Ansible playbook:

```{%raw%}
---

- name: ensure openhab container is running
  hosts: all
  vars:
    container_state: running
    container_name: openhab
    container_image: openhab/openhab:2.3.0-amd64-alpine
    container_dir_config: openhab_conf
    container_dir_data: openhab_userdata
    container_dir_addons: openhab_addons
    container_dir_owner: 9001
    container_dir_group: 9001
    container_run_args: >-
      --rm
      -p 8083:8080/tcp
      -p 8101:8101/tcp
      -p 5007:5007/tcp
      -v "{{exported_container_volumes_basedir}}/{{container_dir_addons}}:/openhab/addons:Z"
      -v "{{exported_container_volumes_basedir}}/{{container_dir_config}}:/openhab/conf:Z"
      -v "{{exported_container_volumes_basedir}}/{{container_dir_data}}:/openhab/userdata:Z"
      --hostname={{ openhab_hostname }}
      --memory=512M
      -e EXTRA_JAVA_OPTS="-Duser.timezone=Europe/Helsinki"
    firewall_port_list:
      - 8083/tcp
      - 8101/tcp
      - 5007/tcp

  tasks:

  - name: ensure container files mount point on host
    tags: mount
    file:
      path: "{{exported_container_volumes_basedir}}/{{ item }}"
      owner: "{{ container_dir_owner }}"
      group: "{{ container_dir_group }}"
      state: directory
      recurse: yes
    with_items:
      - "{{container_dir_addons}}"
      - "{{container_dir_config}}"
      - "{{container_dir_data}}"

  - name: ensure container state
    tags: container
    import_role:
      name: podman_container_systemd
{%endraw%}
```

For some cases I mount NFS storage from my [FreeNAS](https://freenas.org/) for
permanent storage. Nextcloud is good sample of such data. It's simple as adding
few lines of ansible to previous playbook:

```{%raw%}
---

- name: ensure nextcloud container is running
  hosts: all
  vars:
    container_state: running
    container_name: nextcloud
    container_image: nextcloud:latest
    nextcloud_nfs_mountpoint: /mnt/nextcloud_data
    nextcloud_www_dir: nextcloud_www
    nextcloud_www_dir_owner: 33
    nextcloud_www_dir_group: root
    container_run_args: >-
      --rm
      -p 8090:80/tcp
      -v "{{exported_container_volumes_basedir}}/{{nextcloud_www_dir}}:/var/www/html:Z"
      -v "{{ nextcloud_nfs_mountpoint }}/data:/var/www/html/data"
      --hostname={{ nextcloud_fqdn }}
      --memory=1024M
    container_firewall_ports:
      - 8090/tcp

  tasks:

  - name: set variable states when container state is running
    set_fact:
      cron_state: present
    when: nextcloud_state == "running"

  - name: set varialbe states when container state is not running
    set_fact:
      cron_state: absent
    when: nextcloud_state != "running"

  - name: Ensure container data mount points
    tags: mount
    file:
      path: "{{ nextcloud_nfs_mountpoint }}"
      state: directory
      - name: ensure container NFS mounts from NAS
        tags: [ mount, nfs ]
        mount:
          src: "{{ nextcloud_nfs_src }}"
          path: "{{ nextcloud_nfs_mountpoint }}"
          fstype: nfs
          opts: rw,rsize=8192,wsize=8192,timeo=14,intr,vers=3
          state: mounted

      - name: ensure nextcloud www files mount point on host
        tags: mount
        file:
          path: "{{exported_container_volumes_basedir}}/{{nextcloud_www_dir}}"
          owner: "{{ nextcloud_www_dir_owner }}"
          group: "{{ nextcloud_www_dir_group }}"
          state: directory

      - name: ensure container state
        tags: container
        import_role:
          name: podman_container_systemd

      - name: ensure cron is available
        package:
          name:
            - cronie
            - crontabs
          state: present
        when: cron_state == "present"

      - name: ensure nextcloud cron get's run every 15 mins.
        tags: cron
        cron:
          name: "nextcloud periodic job"
          minute: "*/15"
          job: podman exec -u www-data nextcloud /usr/local/bin/php -f /var/www/html/cron.php
          state: "{{ cron_state }}"
{%endraw%}
```

# Get that container finally running, will you!?

I run those playbooks from my AWX web interface. Getting them there is another
story. Here's one liner how to install and make above container to run on given
host:

```
ansible-playbook -b -i myhost.mynet, -e container_state=running run-container-nextcloud-podman.yml
```

and if want tear container down from the host, I do:

```
ansible-playbook -b -i myhost.mynet, -e container_state=absent run-container-nextcloud-podman.yml
```

The podman_container_systemd **will also update the container images** on
consecutive runs. So updating my nextcloud is now just matter of making sure
image is set to use nextcloud:latest, and rerunning the ansible playbook. In my
case, I press a button from AWX using my mobile browser. Easy.

# Controlling the start and stop of containers after install

How about if my container dies? Or if I want to shut it down? That's the part
of using systemd. First of all, podman_container_systemd by default sets
automatic restart for the container. So if container dies due e.g. software bug,
systemd automatically will restart it. Same with reboot, systemd makes sure
container is spawn up.

If I manually want to start or stop the container, I use systemd like with any
other systemd units:

```
sudo systemctl stop nextcloud-container-pod.service
```

and

```
sudo systemctl start nextcloud-container-pod.service
```

# Next steps

Here we learned how to run or remove a container, and set manage firewall for
it, and do updates for container images. This is nice for simple containers. But
this is just part one. In the part two I tell you how to do "docker compose"
-kind of setup of several containers. An example for such use case would be
running e.g. AWX having awx_task, awx_web, memcached, postgres and rabbitmq
containers in one pod defined by kubernets yaml syntax.

Also, I'd be curious to convert these containers to be run as user. But that's
also next step, and honestly I wait for podman having ``` --userns=auto```
switch. I believe it won't take long, the development team is amazing and fast
moving.

Happy containerising until that!

BR,
ikke
