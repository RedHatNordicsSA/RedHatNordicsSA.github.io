---
layout: post
title:  "Automate Podman Containers with Ansible 2/2"
date:   2019-03-25 16:00:00 +0100
tags:
  - containers
  - ansible
author: ikke
---

<p><banner_h>The Challenge</banner_h></p> Now that we know how to run single
container with Podman ([part
1/2](https://redhatnordicssa.github.io/ansible-podman-containers-1)) , it's time
to find out how we can run several containers in a pod, like docker-compose did,
and how kubernetes places containers into sigle pod. We use Ansible and Podman
for that.

![podman-logo](https://github.com/containers/libpod/raw/master/logo/podman-logo.png){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![ansible-logo](assets/images/ansible_logo.png){:height="80px"}

Wouldn't it be wonderful to run Ansible AWX ([Tower upstream](https://www.ansible.com/products/tower)) server like this:

![awx-stack](/assets/images/awx-stack.png)

With only this little amount of automation code (run-aws playbook):

```
{%raw%}
- name: run AWX on host
  hosts: all
  tasks:
    - name: import awx_pod role to install it all
      vars:
        awx_admin_user: admin
        awx_admin_password: foobar
        awx_data_volume_host_path: /tmp/awx_data
        awx_db_volume_host_path: /tmp/awx_db
        awx_host_port: 8052/tcp
        #container_state: absent or running
      import_role:
        name: awx_pod
{%endraw%}
```

Well, good news is that you can do it, almost just simple as that :)

# Use case?

Personally I just wanted as easy as possible way of running several containers
at once to provide a service. I run AWX (Ansible Tower upstream) at home for my
personal stuff, and I thought it would make good sample case to this blog.

Industry use case could be you produce and run containers in cloud using
OpenShift Container Platform. But you need in some cases run parts of
application in traditional servers. Think of pushing parts of stack into e.g.
industrial plant in Industrial Internet of Things (IIoT) case.

For this exercise I checked what [AWX docker installer](https://github.com/ansible/awx/blob/devel/installer/roles/local_docker/tasks/standalone.yml) does. I converted it to Podman commands. It requires steps:

1. start pod
2. insert postgres containers
3. insert rabbitmq container
4. insert memcached container
5. insert awx_tasks container
6. insert awx_web container

Some of those containers need storage from host to survice updates and restarts,
and pod needs to have port 80 (www) exposed from awx_web container. Pod get's
created by command ```podman pod create awx``` and containers are inserted to
awx pod by  ```podman run -dt --pod awx ...```, like you see from Brent's blog.

I got all that done by running Ansible command module 6 times. But I find it
way nicer to control such complicated stack by single kubernetes yaml
configuration file instead.

I have based my Ansible playbooks on steps described in excellent Podman blogs
by Brent Baude. See the first two links at the bottom.

# Describing pod and containers in single yaml

After reading Brent's second blog from list at bottom, I thought that's the way
how I want to manage pods and containers. One clear yaml file which can be
templated for ansible, and that file passed as parameter for systemd pod service
file. That way it keeps clean and simple.

Here is the architecture of running AWX in one pod, using separate containers
for different services.

![awx-pod](/assets/images/awx-pod.png)

Here's snipplet of my awx.yml defining the pod. Pay attention how you can define
host mount points for persistent storage, and just keep listing containers
with unique settings. There could be CPU/mem limits, privilege escalation and
what ever variables you are used to set to any container.

```
{%raw%}
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: {{awx_pod_label}}
  name: {{awx_pod_name}}
spec:
  #
  # define exported volumes for permanent data
  #
  volumes:
  - name: awx-data-volume
    hostPath:
      path: {{awx_data_volume_host_path}}
      type: Directory
  - name: db-volume
    hostPath:
      path: {{awx_db_volume_host_path}}
      type: Directory
  containers:
  #
  # postgres container
  #
  - command:
    - docker-entrypoint.sh
    - postgres
    env:
    - name: PATH
      value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/{{awx_postgres_version}}/bin
    - name: POSTGRES_USER
      value: {{awx_postgres_user}}
    - name: POSTGRES_DB
      value: awx
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
    - name: POSTGRES_PASSWORD
      value: {{awx_postgres_password}}
    image: docker.io/library/postgres:{{awx_postgres_version}}
    name: postgres
    volumeMounts:
    - mountPath: /var/lib/postgresql/data/pgdata:z
      name: db-volume
  #
  # memcached container
  #
  - command:
    - docker-entrypoint.sh
    - memcached
    env:
    image: docker.io/library/memcached:alpine
    name: memcached
  #
  # awx-web container
  #
  {%endraw%}
  ...
```

See the [full file
here](https://github.com/ikke-t/awx_pod/blob/master/templates/awx.yml.j2).
Kubernetes yaml is easy to read and understand, if you are into containers.
Also, bonus being there are tons of examples running stuff on kubernetes, and
you can snatch the configuration from such example also. This is really close to
kubernetes configuration file format.

# How does this yaml get used by systemd?

Let's look at the Ansible playbook and roles organization. Following picture
shows what happens behind the run-aws.yml playbook you saw in the first chapter.

![awx-pod](/assets/images/awx-ansibles.png)

You see from above run-aws.yml depends on two Ansible roles:

1. [aws_pod](https://github.com/ikke-t/awx_pod)
2. [podman_container_systemd](https://github.com/ikke-t/podman-container-systemd)

After writing the first blog of this series, I added new role 'awx_pod' for
creating awx.yml kubernetes file, and pass it to 'podman_container_systemd'.
It also creates the needed directories for exported volumes and creates list
of required container images. See the [awx_pod task list here](https://github.com/ikke-t/awx_pod/blob/master/tasks/main.yml),
it's light weight.

Both of those are in galaxy and you need to install them before running
run-awx.yml.

# How did podman_container_systemd change?

I modified my podman_container_systemd -ansible role (see [part
1/2](https://redhatnordicssa.github.io/ansible-podman-containers-1))
to handle yaml files for Podman containers. I also added download for container
images.

```
{%raw%}
- name: seems we use several container images, ensure all are up to date
    command: "podman pull {{ item }}"
    when: container_image_list
    with_items: "{{ container_image_list }}"

  - name: if running pod, ensure configuration file exists
    stat:
      path: "{{ container_pod_yaml }}"
    register: pod_file
    when: container_pod_yaml is defined
  - name: fail if pod configuration file is missing
    fail:
      msg: "Error: Asking to run pod, but pod definition yaml file is missing: {{ container_pod_yaml }}"
    when:
      - container_pod_yaml is defined
      - not pod_file.stat.exists

  - name: "create systemd service file for pod: {{ container_name }}"
    template:
      src: systemd-service-pod.j2
      dest: "{{ service_files_dir }}/{{ service_name }}"
      owner: root
      group: root
      mode: 0644
    notify:
      - reload systemctl
      - start service
    register: service_file
    when: container_image_list is defined
{%endraw%}
```

And this is how simple [systemd service file](https://github.com/ikke-t/podman-container-systemd/blob/master/templates/systemd-service-pod.j2) is now, as parameters are in yaml:

```
{%raw%}
ExecStart=/usr/bin/podman play kube {{ container_pod_yaml }}
{%endraw%}
```

# Shut up, and run the pod!

Finally, really, this is all that is required to ask ansible to run AWX:

```
{%raw%}
mkdir roles
cat >>roles/requirements.yml<<EOF
---
- src: ikke_t.awx_pod
  name: awx_pod
- src: ikke_t.podman_container_systemd
  name: podman_container_systemd
EOF

ansible-galaxy --roles-path roles install -r roles/requirements.yml
ansible-playbook -i my-awx-host, -b run-awx.yml
{%endraw%}
```

This is what the containers in awx pod look like:

```
$ sudo podman ps -a
CONTAINER ID  IMAGE                                 COMMAND               CREATED         STATUS             PORTS                   NAMES
b715ccd5af0b  docker.io/ansible/awx_rabbitmq:3.7.4  docker-entrypoint...  38 seconds ago  Up 36 seconds ago  0.0.0.0:8052->8052/tcp  rabbitmq
804db65e2e00  docker.io/ansible/awx_task:latest     /tini -- /bin/sh ...  38 seconds ago  Up 36 seconds ago  0.0.0.0:8052->8052/tcp  awxtask
d062d528dd4b  docker.io/ansible/awx_web:latest      /tini -- /bin/sh ...  38 seconds ago  Up 37 seconds ago  0.0.0.0:8052->8052/tcp  awxweb
19a7ee0545eb  docker.io/library/memcached:alpine    docker-entrypoint...  38 seconds ago  Up 37 seconds ago  0.0.0.0:8052->8052/tcp  memcached
ef70a60c135b  docker.io/library/postgres:9.6        docker-entrypoint...  38 seconds ago  Up 37 seconds ago  0.0.0.0:8052->8052/tcp  postgres
cf2f04239b1c  k8s.gcr.io/pause:3.1                                        38 seconds ago  Up 37 seconds ago  0.0.0.0:8052->8052/tcp  8fff646c1b7c-infra
```

And now I am ready to login to my AWX!

![awx-screenshot](assets/images/awx-login.png)

We are done!

# Some notes

* I used upstream Podman here. It's not released yet, it was this version:
https://kojipkgs.fedoraproject.org//packages/podman/1.2.0/24.dev.git0458daf.fc31/x86_64/podman-1.2.0-24.dev.git0458daf.fc31.x86_64.rpm. Be patient, this all will likely be in
  normal package repositories soon.
* Even if I use AWX for this hobby stuff, remember it's upstream project, and
  break at any time and likely gets your dog pregnant. Use Tower for production.
* I still need to understand selinux labels better for the exported volumes.
  It perhaps requires still some development effort. Hopefully it's user error
  :)
* If you want to remove all above installed stuff, run
  ```ansible-playbook -i my-awx-host, -b -e container_state=absent run-awx.yml```

# Links to references

I found the following blogs excellent. Also, the Podman team is amazingly
helpful, active in github and IRC to fix bugs and RFEs. Thank you guys!

* [Podman: Managing pods and containers in a local container runtime, By Brent Baude](https://developers.redhat.com/blog/2019/01/15/podman-managing-containers-pods/)
* [Podman can now ease the transition to Kubernetes and CRI-O, By Brent Baude](https://developers.redhat.com/blog/2019/01/29/podman-kubernetes-yaml/)
* [Podman and user namespaces: A marriage made in heaven, Dan Walsh](https://opensource.com/article/18/12/podman-and-user-namespaces)
* And there are may other [blogs about Podman Red Hat developer portal](https://developers.redhat.com/search/?t=podman)


BR,
ikke
