---
layout: post
title:  "Test Ansible Roles using Molecule and Podman"
date:   2020-5-14 12:00:00 +0200
tags:
  - containers
  - ansible
  - molecule
author: ikke
---

<p><banner_h>The Challenge: Lightweight and easy testing</banner_h></p>
I needed to have testing added to Ansible roles. There are various people in
the customer organization developing roles, and we want a lightweight, easy
to use test process to unify the looks and quality of the roles.

# Solution

Meet [Molecule][molecule-docs] with [Podman][podman] plugin and
[Ansible][ansible] as test language!

![molecule-logo](https://github.com/RedHatNordicsSA/RedHatNordicsSA.github.io/raw/master/assets/images/molecule-logo.png){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![podman-logo](https://github.com/containers/libpod/raw/master/logo/
podman-logo.png){: height="80px"}
![plus](assets/images/plus.png){:height="40px"}
![ansible-logo](assets/images/ansible_logo.png){:height="80px"}

I created also a short video about this as a support material:

<iframe width="560" height="315" src="https://www.youtube.com/embed/2KyLePKsU-E" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

# Benefits of podman for containers and ansible for tests

* Containers are slim and quick
* Podman requires no root, which makes it easily part of CI/CD
* Easy testing language

We are testing ansible roles by the people who write ansible roles, so writing
test cases couldn't get simplier than writing them in ansible too! And since
molecule has developed since [Peter's earlier blog using VMs][pb], now we can
use podman as molecule plugin to spin up tests quickly in containers. And thanks
to podman, no root gets hurt during the process. Which means we can put this
safely into the CI/CD process, because of everything gets run as a user.

This hands on is done using RHEL8 Linux. However, you can run different
container images for testing the different Linuxes. Also, if Solaris etc... is
needed, you need to run those cases in VMs like Peter explained earlier. You
can however use ansible as a test language, instead of [Test Infra][ti].

# Wait, isn't there quite much to put into one blog?

Sure, and molecule has been blogged a lot about already as well as ansible and
podman. So instead of going into those, check out e.g. these blogs about them:

* [Practical Ansible Testing with Molecule, Fabian von Feilitzsch][practical] is
  good read to understand the molecule tree structure
* [Jeff Geerling's two Vlogs][geerling vlog] about Ansible and Molecule are
  great, and I recommend you to have his book
  [Ansible for DevOps][geerling devops], it's there in written.

In this blog I shortly describe the process of running all this in podman on
RHEL8, as that's the use case at hand. I trust you learn molecule and ansible
from elsewhere. By making this public, I hope you can reuse it in your work.

# Let's go through the process

We use RHEL8 in the guide, along with [RHEL8 UBI container images][ubi8]. RHEL
is set up with required repos, [see ansible playbook][prepare rhel8].

## Install Podman, Ansible and required tools for python

```
sudo dnf install -y podman ansible python3 python3-virtualenv gcc git
```

## Install Molecule

Create a python virtual environment where we install all molecule tools. We use
pip to install python modules, and python virtual environment to keep them
separate from system modules.

```
python3 -m virtualenv molecule_ansible
source molecule_ansible/bin/activate
pip install ansible testinfra molecule podman python-vagrant ansible-lint \
  flake8 molecule[lint] molecule[podman]
```

## Apply molecule to ansible role

Activate molecule virtual environment if already didn't:

```
source molecule_ansible/bin/activate
```

Create a new role by using molecule to get the basic molecule files in place.
This will run ansible-galaxy init role in background, and will add the initial
molecule config and test files.

```
molecule init role my-new-role --driver-name podman
```

If there is an existing ansible role, add just the molecule files:

```
molecule init scenario -r ftp --driver-name podman
```
Note, I've done an example which has the molecule set, get it with:

```git clone https://github.com/RedHatNordicsSA/molecule-podman-blog.git```

Read the following chapters and play with
it by modifying the settings and test cases.

## Molecule

Now let's get going. Investigate how:
* container gets prepared for the test in the [converge.yml file][converge]. In
  this very simple case it includes the role to handle FTP.
* molecule is set up to use podman, and a container that has init in it.
  This is in the [molecule.yml file][molecule yml].
* tests are written in ansible in [verify.yml file][verify yml].

### Molecule options

Notice how the podman is defined as driver, and init container is set with
required options, and ansible is set to be the test tool:

```
driver:
  name: podman
platforms:
  - name: instance
    image: ubi8/ubi-init
    pre_build_image: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    privileged: true
    command: "/usr/sbin/init"
provisioner:
  name: ansible
verifier:
  name: ansible
```

You can see the selected tests also in the molecule.yml.

### Ansible test cases

Test cases are trivial to write with ansible. In this case it just runs
install and deactivate commands for the service in check-only mode, and
verifies if it would have needed to apply changes. In our case a change would
show a failure, as everything should be already setup. So it's ok if nothing
would have needed a change.

```
    - name: check if vsftpd is installed
      package:
        name: vsftpd
        state: present
      check_mode: yes
      register: pkg

    - name: fail if package was not installed
      assert:
        that:
          - pkg.changed is false
        fail_msg: "Package vsftpd was not installed!"
        success_msg: "Package vsftpd was installed."
```

## Podman

You don't need to know about podman in this case. I'd only add image pruning to
be done once in a while to the test machine, if it's a permanent VM. You could
be running these tests in Ansible Tower or Jenkins machine, in which case you
want to prune the images. Remember, no root required, so podman gives you a lot
of options where to run it.

Here's how it looks like when the container is started step by step. Podman has
pulled the image, and is running a test container. You can do only those parts
by the folliwing commands:

```
(molecule_ansible) [cloud-user@localhost ftp]$ molecule converge
(molecule_ansible) [cloud-user@localhost ftp]$ podman ps
CONTAINER ID  IMAGE                                            COMMAND         CREATED         STATUS             PORTS  NAMES
712c7b4ac78c  registry.access.redhat.com/ubi8/ubi-init:latest  /usr/sbin/init  33 seconds ago  Up 33 seconds ago         instance
(molecule_ansible) [cloud-user@localhost ftp]$ molecule login
[root@712c7b4ac78c /]# whoami
root
[root@712c7b4ac78c /]# exit
(molecule_ansible) [cloud-user@localhost ftp]$ molecule destroy
```

# Executing the test

First and foremost, this is enough to run the tests:

```
(molecule_ansible) [cloud-user@localhost ftp]$ molecule test
```

In our case it has a happy ending:

```
    TASK [check if vsftpd is installed] ********************************************
    ok: [instance]

    TASK [fail if package was not installed] ***************************************
    ok: [instance] => {
        "changed": false,
        "msg": "Package vsftpd was installed."
    }

    TASK [check service is stopped] ************************************************
    ok: [instance]

    TASK [fail if service was activated] *******************************************
    ok: [instance] => {
        "changed": false,
        "msg": "Service vsftpd was disabled."
    }

    TASK [test result] *************************************************************
    ok: [instance] => {
        "msg": "FTP daemon was installed and disabled. Test OK"
    }

    PLAY RECAP *********************************************************************
    instance                   : ok=6    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

See [Molecule command documentation][molecule-usage] for details.

## Additional Molecule commands

These commands are all executed by the ```molecule test```, but sometimes it's
handy to run them separately. E.g. when writing the tests, or debugging failed
tets.

* ```molecule lint```: check the code syntax
* ```molecule create```: create container for the test setup
* ```molecule list```: check the container is running
* ```molecule converge```: run the tests in the running container
* ```molecule idempotence```: test idempotence by running the role in loop
* ```molecule login```: execute shell into test container to inspect it
* ```molecule destroy```: delete the test container

# Conclusion

It is really convenient to run tests in light weight manner. I do not go into
CI/CD tooling in this case. My customers are often in disconnected environments
where they have each different set of tools. This method should be applicable
to adopt to many of them.

Good luck in testing, may the tests be successful for you!

BR, Ilkka

[podman]: https://podman.io/
[ansible]: https://www.ansible.com/
[pb]: https://redhatnordicssa.github.io/how-we-test-our-roles
[ti]: https://testinfra.readthedocs.io/en/latest/
[ubi8]: https://www.redhat.com/en/blog/introducing-red-hat-universal-base-image
[molecule-docs]: https://molecule.readthedocs.io/en/latest/
[molecule-usage]: https://molecule.readthedocs.io/en/latest/usage.html
[test-driven]: https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/
[practical]: https://www.ansible.com/hubfs//AnsibleFest%20ATL%20Slide%20Decks/Practical%20Ansible%20Testing%20with%20Molecule.pdf
[geerling vlog]: https://www.youtube.com/watch?v=FaXVZ60o8L8
[geerling devops]: https://leanpub.com/ansible-for-devops
[prepare rhel8]: https://github.com/ikke-t/vagrant/blob/master/ansibles/prepare_rhel8.yml
[converge]: https://github.com/RedHatNordicsSA/molecule-podman-blog/blob/master/ftp/molecule/default/converge.yml
[molecule yml]: https://github.com/RedHatNordicsSA/molecule-podman-blog/blob/master/ftp/molecule/default/molecule.yml
[verify yml]: https://github.com/RedHatNordicsSA/molecule-podman-blog/blob/master/ftp/molecule/default/verify.yml
