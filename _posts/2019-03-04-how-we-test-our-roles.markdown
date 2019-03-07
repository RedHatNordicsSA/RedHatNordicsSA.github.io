---
layout: post
title:  "Testing Ansible automation with molecule"
date:   2019-03-04 12:30:00 +0200
tags: random
author: pgustafs
---


<p><banner_h>The Challenge</banner_h></p>
We are heavily believers in that you should store your server infrastructure in a Git repository, so we decided to live as we preach! **Everything that we install in our Demo environment should be automated with Ansible** and versioned by Git. This to:
* Save **time** when setting up a demo environment
* Fight the demo ghost - we need our demos to be **consistent** each time we deploy them
* Being able to **reproduce** our infrastructure on new platforms(physical/virt/cloud) from time to time  
* Get **traceability** on what changes have been made
* Get an **immutable** infrastructure - instead of troubleshooting simply create new infrastructure components

This is quite cool since we now are describing our whole infrastructure with code and all our code are versioned by git
([Infrastructure-as-Code[IaC]](https://en.wikipedia.org/wiki/Infrastructure_as_code)). **So far all good**. We started to collaborate across our Nordic team to created re-usable Ansible roles that we could use in our different automation workflows, but we **did not** implement a way to automatically and constantly execute and test our Ansible code, which lead to:
* Many late nights
* Roles got outdated, **code that is not automatically and constantly executed and tested will eventually decompose sooner or later!**
* Roles did not work with new versions of Ansible engine(this is totally fine, if it is detected before someone upgrades Ansible on our Tower server :rage: and you detect it **one hour before a demo** when trying to deploy the environment)
* Due to time we did not always "read **never**" develop our Ansible roles in a consistent way
* Changes made to one Ansible role could destroy for another
* We also saw that people hesitated to do changes to others roles, since they were afraid of break them
* Among more....
<div align="center">
<div markdown="1">
![](/assets/images/changinstuff-big.png){:height="50%" width="50%"}
</div>
</div>

<p><banner_h>The Solution</banner_h></p>

**Lesson learned**, the glory days of **run it, watch it fail, run it again** is over. **We´re software developers now** and must start to threat our code as **code**, let's peek over the bulletproof wall to the dev guys & girls in the software development team and see if we can use something they have used for years? Hmm it seems like the **Test-driven development[TDD]** and **Continuous Integration[CI]** methodologies would solve our problems, right?. Great, is there any solid Open Source **Testing framework** for Ansible available ? It sure is, **September 26th, 2018** Ansible announced that they will adopt two new projects **molecule** and **ansible-lint**, which are great tools that now [Red Hat](http://redhat.com) intends to invest resources working with the community to make them even better. **kudos** to [Cisco](http://cisco.com) who transferred the molecule project over to the Red Hat Ansible team.
<p align="center">
  <img src="https://raw.githubusercontent.com/ansible/molecule/master/asset/molecule.png" width="256" title="Molecule Logo">
</p>
**[Molecule](https://github.com/ansible/molecule)** is the official testing framework for Ansible roles. It provides a streamlined way to create a virtualized environment to test the syntax and functionality of a role.
From the Molecule docs:
>Molecule encourages an approach that results in consistently developed roles that are well-written, easily understood and maintained.

**[Molecule](https://github.com/ansible/molecule)**:
* Handles Project linting by invoking configurable linters (yamlling, ansible-lint)
* Executes our roles in a defined provider (OpenStack, Docker, Vagrant, etc...)
* Handle role testing by invoking configurable verifiers (Testinfra)
* Test against multiple versions of Ansible (Tox)
* Now part of the Ansible project
* pip installable - entire dev/test environment can happily live in a virtualenv

**[Testinfra](https://testinfra.readthedocs.io/en/latest/)**:
With Testinfra you can write unit tests in Python to test the actual state of your servers configured by Ansible. This is really great when you are starting to do test-driven development of your roles. Normally when developing an Ansible role/playbook
you start writing the Ansible tasks, but now with this you are thinking ahead about tests and starts with writing the unit tests for your Ansible role instead. **Time spent here is well invested time**, especially if you are part of a big team that maintains many roles.   

**[Tox](https://tox.readthedocs.io/en/latest/)**:
Tox is a generic Python virtualenv management and test command line tool we use for test our roles against multiple versions of Ansible. As soon as a new version of Ansible engine is released we can automatically test all our roles against it.

In this post I will focus on how you can use Molecule to test-driven development of your Ansible roles and in the next post I will add some CI to it.  

<p><banner_h>Setup the development system</banner_h></p>
## Preparing your environment for installation
Register your system with the Red Hat Content Delivery Network(You can skip this step if you use centos):
```
sudo subscription-manager register
```

Attach your subscription to the system:
```
sudo subscription-manager attach --pool=pool_id
```

Enable the required repositories:
```
sudo subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms.
```

## Drivers and Platforms
When testing your roles Molecule will spin up one or more instances onto it will execute your role.
Molecule uses drivers for this, which let you choose the backend for your platforms. Docker, Vagrant, AWS, Google Cloud, etc. are all [drivers](https://molecule.readthedocs.io/en/latest/configuration.html?highlight=drivers). Platforms are the instances created using drivers. Each container or VM is a different platform that your code will be tested on.

To make it simple, Molecule uses Ansible to manage the instances it operates on so basically Molecule support any provider that Ansible supports. In this guide i will use two different providers, **Docker** (default driver) and Vagrant backed by KVM.

### The Docker driver
Docker is the default driver for Molecule, It's lightweight since containers have minimal footprint. And with the wide choice of container images available, you can easily test your Ansible role on different platforms like RHEL, SUSE, ubuntu etc. But since containers are what they are, (a convenient way to package applications) they are **not suitable for all use case**. They're no VM no matter how many workarounds and hacks you come up with like systemd in a container, bind-mounting host sockets and volumes, privileged containers and so on. So they definitely have their place but it really depends on the kind of role you're writing and testing. Generally, things like collecting info from the target platform, talking to APIs, ensuring config files are compliant, etc. are good candidates to be tested using Docker. On the other hand deploying applications, dealing with services, configuring networking, etc. are almost always best to test on VMs.

### Testing for RHEL platform
There's another challenge with using Docker if the role is meant for the RHEL platform. That is, due to the required entitlements, one can only run RHEL container images on a subscribed RHEL system. This is not a problem for me since I am using RHEL as OS on my development system, but it can be if you are using for example CentOs, Fedora or OSX.

### Install Docker
Install docker packages:
```
sudo yum install docker device-mapper-libs device-mapper-event-libs
```
Start docker:
```
sudo systemctl start docker.service
```
Enable docker:
```
sudo systemctl enable docker.service
```
Check docker status:
```
sudo systemctl status docker.service
```

### The Vagrant driver
Vagrant is currently the only driver for using local VMs with Molecule in a simple way. Vagrant in turns leverages different providers, like VirtualBox or Libvirt, to actually create the VMs.

### Install Vagrant
See following blog post [Install Vagrant and Libvirt (KVM) on RHEL](https://redhatnordicssa.github.io/install-Vagrant-and-libvirt-KVM-on-on-rhel)

## Install Molecule
[pip](https://pip.pypa.io/en/latest/usage/) is the only supported installation method for Molecule so I will use Python Virtual Environments to install molecule and its requirements onto my system, using Python Virtual Environments also makes it possible for me to install several different versions of Ansible onto the same system to test my roles with. **Note** that Molecule requires at least Ansible version **2.4**.

In the world of Python, an **virtual environment** is a folder (directory) which contains everything that a Python project (application) needs in order to run in an isolated fashion. When it is initiated, it automatically comes with its own Python interpreter - a copy of the one used to create it - alongside its very own pip.

Install the Python virtualenv package:
```
sudo yum install python-virtualenv.noarch
```
Create/Initialize a Python virtual environment (virtualenv) containing the Python2.7 interpreter. For this purpose I name my virtualenvs after the ansible version I install in them:
```
cd ~
python2.7 -m virtualenv molecule_ansible2.7
```
Activate the virtualenv:
```
source ~/molecule_ansible27/bin/activate
```
Install Molecule, Testinfra, Ansible and the drivers into the virtualenv:
```
pip install ansible testinfra molecule docker python-vagrant
```
Repeat above steps for all the Ansible versions you want to test your roles with or create them all in one chunk:
```
for i in 2.4 2.5 2.6 2.7
do
python2.7 -m virtualenv molecule_ansible$i
~/molecule_ansible$i/bin/pip install ansible==$i.* testinfra molecule docker python-vagrant
done
```
To deactivating a virtual environment:
```
deactivate
```

<p><banner_h>Lets get started</banner_h></p>
<notify_green><p markdown="1">**Now when we have our environment setup, lets develop and test an Ansible role using Molecule!!!**</p></notify_green>
## Scenario
Molecule treats scenarios as a first-class citizens, A scenario is a self-contained directory containing everything necessary for testing the role in a particular way. You can for example create one scenario using Docker for testing the role and one scenario using KVM for testing the role and so on. When initializing a new role with Molecule an default scenario using Docker is created.   

### Initializing a new role with Molecule
Below command uses ansible-galaxy behind the scenes to generate a new Ansible role, then it injects a molecule directory in the role, and sets it up to run builds and test runs in a docker container.
```
molecule init role -r ansible-role-apache
```
Layout:
```
|-- defaults
|   `-- main.yml
|-- handlers
|   `-- main.yml
|-- meta
|   `-- main.yml
|-- molecule
|   `-- default
|       |-- Dockerfile.j2
|       |-- INSTALL.rst
|       |-- molecule.yml
|       |-- playbook.yml
|       `-- tests
|           |-- test_default.py
|-- README.md
|-- tasks
|   `-- main.yml
`-- vars
    `-- main.yml
```

### Initializing a new role using rhnordicssa skeleton
The rhnordicssa skeleton provides a standardized starting point for role development and molecule configuration for our team that is made to work with the ansible-galaxy CLI.
we chose to create a skeleton for this to make our roles more consistent written across the team and it also saves us a lot of time when start writing a new role.
The skeleton comes with two different scenarios configured and ready to use, one using Docker and one using Vagrant(KVM).
```
git clone https://github.com/RedHatNordicsSA/meta_skeleton.git
ansible-galaxy init --role-skeleton=meta_skeleton ansible-role-apache
```
Layout:
```
|-- AUTHORS
|-- defaults
|   `-- main.yml
|-- files
|-- handlers
|   `-- main.yml
|-- LICENSE
|-- meta
|   `-- main.yml
|-- molecule
|   |-- docker
|   |   |-- create.yml
|   |   |-- destroy.yml
|   |   |-- Dockerfile.j2
|   |   `-- molecule.yml
|   |-- kvm
|   |   `-- molecule.yml
|   `-- shared                 <--- This dir contains files used by both scenarios
|       |-- playbook.yml
|       |-- prepare.yml
|       `-- tests
|           `-- test_default.py
|-- README.md
|-- tasks
|   `-- main.yml
|-- templates
|-- tests
|   `-- yamllint.yml
`-- vars
    `-- main.yml
```

## Important files
\- <c>INSTALL.rst</c>: Instructions on how to install the dependencies for the driver in use.  
\- <c>molecule.yml</c>: The Molecule settings for the role. what driver to use, what OS to use, how to lint your role, what tests to run, etc.  
\- <c>playbook.yml</c>: The converge playbook that will run the role. This should be configured with any custom variables and tasks needed for the role to work. Additional post-tasks can also be added for verifications.  
\- <c>prepare.yml</c>: Some drivers will have a preparation playbook that uses the raw module to install Python.  
\- <c>tests/test_default.py</c>: Basic Testinfra test, can be extended. If the you are not going to write any Python tests, the "molecule/<scenario>/tests" directory should be removed entirely and have the test disabled in **molecule.yml**.

<notify><p markdown="1">For the remaining of this post I will use the **ansible-role-apache** role initialized from the **[rhnordicssa skeleton](https://github.com/RedHatNordicsSA/meta_skeleton)** with two different scenarios configured, **Docker** and **KVM**. If you'd like to see the example code, you can check out the **[GitHub repository](https://github.com/RedHatNordicsSA/ansible-role-apache)**.</p></notify>

### Configure Molecule (molecule.yml)
Lets take a closer look at the Molecule configuration for the **kvm** scenarion <c>molecule/kvm/molecule.yml</c>:
```yaml
dependency:
  name: galaxy
driver:
  name: vagrant
  provider:
    name: libvirt
    type: libvirt
    options:
      memory: 1024
      cpus: 1
lint:
  name: yamllint
  options:
    config-file: tests/yamllint.yml
platforms:
  - name: instance1
    box: centos/6
    groups:
      - rhel
  - name: instance2
    box: centos/7
    groups:
      - rhel
provisioner:
  name: ansible
  log: true
  lint:
    name: ansible-lint
  playbooks:
    prepare: ../shared/prepare.yml
    converge: ../shared/playbook.yml
scenario:
  name: kvm
verifier:
  name: testinfra
  directory: ../shared/tests
  options:
    # Add a -v so you see the individual test names,
    # particularly useful with parameterized tests
    v: true
    sudo: true
  lint:
    name: flake8
  # Using the shared directory is useful for sharing tests across scenarios,
  # but is not a requirement. For scenario specific tests, add the appropriate
  # file path to the test or test directory below
  additional_files_or_dirs:
    - ../shared/*
```
I'll walk you through some parts of the configuration, but the full list of configuration options can be found [here](https://molecule.readthedocs.io/en/latest/configuration.html).

Below tells Molecule to use **galaxy** to install dependencies for the role, alternate dependency managers to use are **gilt** and **shell**:
```yaml
dependency:
  name: galaxy
```
Below tells Molecule to use the **vagrant** driver to create two virtual instances in libvirt, one with **centos 6** named **instance1** and one with **centos 7** named **instance2**. Both instances are configured with **1Gb** of ram and **one CPU** and added to the group **rhel**. **groups** is optional, but I am using them a lot when testing multi-tier playbooks:   
```yaml
driver:
  name: vagrant
  provider:
    name: libvirt
    type: libvirt
    options:
      memory: 1024
      cpus: 1
platforms:
  - name: instance1
    box: centos/6
    groups:
      - rhel
  - name: instance2
    box: centos/7
    groups:
      - rhel
```
Below tells Molecule to use Ansible for provisioning the instances on which your role will be tested on and it points out <c>molecule/shared/playbook.yml</c> as the converge playbook. **Note** that I do not store the converge playbook under the <c>molecule/scenatio_name/</c> directory, because I am using the same playbook for all scenarios and don't want to maintain several copies of it:
```yaml
provisioner:
  name: ansible
  log: true
  lint:
    name: ansible-lint
  playbooks:
    prepare: ../shared/prepare.yml
    converge: ../shared/playbook.yml
scenario:
  name: kvm
```
### Execute your first Molecule test
Lets execute a full test using **KVM** as the target platform, I know we have not created any Ansible code yet, so all test should pass with green color.
```
source ~/molecule_ansible27/bin/activate
cd ansible-role-apache
molecule test --scenario-name kvm
```
Molecule now runs through **all** the testing steps(stages):
![](/assets/images/molecule_flow.png){:height="100%" width="100%"}
<notify>
<div markdown="1">
The order of events for tests and which steps that will be executed can be changed from the **defaults** in <c>molecule.yml</c>:
```yaml
scenario:
  name: default
  create_sequence:
    - create
    - prepare
  check_sequence:
    - destroy
    - dependency
    - create
    - prepare
    - converge
    - check
    - destroy
  converge_sequence:
    - dependency
    - create
    - prepare
    - converge
  destroy_sequence:
    - destroy
  test_sequence:
    - lint
    - destroy
    - dependency
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - side_effect
    - verify
    - destroy
```
</div>
</notify>
### Working with Molecule
Molecule is great for testing your roles but it is also a great tool while developing you role, instead of executing a full test using <c>molecule test -s kvm</c> you can execute <c>molecule converge -s kvm</c>: which will create the virtual environment, run the test playbook.yml and leave the environment running. This is great cause then molecule don't have to re-create your test environment each time you want to test a new change during the development of the role, it will just run the playbook on the existing environment.

This is usually how the flow looks like for me when writing a new role or modifying an existing role:
```
git clone https://github.com/RedHatNordicsSA/meta_skeleton.git
ansible-galaxy init --role-skeleton=meta_skeleton ansible-role-apache
molecule converge -s kvm
<add/change some tasks>
molecule converge -s kvm
<Fix the tasks that failed, if any>
molecule converge -s kvm
<Wow the playbook runs without errors, lets execute it one more time to verify that
the role is idempotent>
molecule converge -s kvm
<Great, no task was marked as changed, lets run our Testinfra tests>
molecule verify -s kvm
<All tests passed, lets clean up and push the code to Git>
molecule destroy -s kvm
git .........
```

You can also login to the virtual environment for troubleshooting:
```
molecule login --host instance1  -s kvm
Warning: Permanently added '192.168.121.32' (ECDSA) to the list of known hosts.
Last login: Wed Mar  6 20:19:15 2019 from 192.168.121.1
[vagrant@instance1 ~]$ sudo su -
[root@instance1 ~]#
```
To get the instance name:
```
molecule list -s kvm
Instance Name    Driver Name    Provisioner Name    Scenario Name    Created    Converged
---------------  -------------  ------------------  ---------------  ---------  ---------
instance1        vagrant        ansible             kvm              true       true
```
### Time to create some code
Normally I would start writing Ansible tasks in <c>tasks/main.yml</c> now. But since we're thinking ahead about tests, let's start creating the Testinfra test instead (otherwise called test-driven development). Since we need a running web server, we've got three requirements:
* Is the httpd package installed ?
* Is the httpd service running ?
* Is web service listening at port 80?

Lets add the following tests to the default Python script that got created for Testinfra while initializing the role <c>molecule/shared/tests/test_default.py</c>:
```python
def test_apache_is_installed(host):
    apache = host.package("httpd")
    assert apache.is_installed

def test_apache_running_and_enabled(host):
    apache = host.service("httpd")
    assert apache.is_running
    assert apache.is_enabled

def test_port_80_is_listening(host):
    socket = host.socket("tcp://80")
    assert(socket.is_listening)
```
We're using three built-in [modules](https://testinfra.readthedocs.io/en/latest/modules.html), Package, Service and Socket, to check the state of the system after the Ansible role executes. The output from molecule while executing the tests will look like this:
```
--> Action: 'verify'
--> Executing Testinfra tests found in /root/ansible-role-apache/molecule/kvm/../shared/tests/...
    ============================= test session starts ==============================
    platform linux2 -- Python 2.7.5, pytest-3.9.3, py-1.7.0, pluggy-0.8.1 -- /root/molecule_lab/bin/python2.7
    rootdir: /root/ansible-role-apache/molecule, inifile:
    plugins: testinfra-1.16.0
collected 3 items                                                              

    ../shared/tests/test_default.py::test_apache_is_installed[ansible:/instance1] PASSED [ 33%]
    ../shared/tests/test_default.py::test_apache_running_and_enabled[ansible:/instance1] PASSED [ 66%]
    ../shared/tests/test_default.py::test_port_80_is_listening[ansible:/instance1] PASSED [100%]

    =========================== 3 passed in 6.00 seconds ===========================
Verifier completed successfully.
```
<notify_green>Above was very simple tests to illustrate the functionality of Testinfra, but in a live environment it's best used for more sophisticated functional tests. We trust Ansible to do what it says it did :)</notify_green>

See following example on a functional test:
```python
def test_mount(host):
    # by using a routable IP address for the NFS mount we are able to test the
    # firewall configuration even from localhost
    ip_addr = host.ansible.get_variables()['ansible_host']
    with host.sudo():
        cmd = host.run('mount {}:/srv/share1 /mnt'.format(ip_addr))
        assert cmd.rc == 0


def test_file_write(host):
    assert host.run('echo "test" > /mnt/test_file.txt').rc == 0
```

**Now we can add our desired tasks and templates to the role to satisfy the requirements:**

Edit <c>defaults/main.yml</c>:
```yaml
# defaults file for ansible-role-apache
apache_package_state: present
apache_service_state: started
apache_service_enabled: true
apache_listen_port: 80
apache_firewall_state: enabled
```
Edit <c>tasks/main.yml</c>:
```yaml
# role tasks
- name: Install Apache.
  package:
    name: httpd
    state: "{%raw%}{{ apache_package_state }}{%endraw%}"

- name: Ensure Apache has selected state.
  service:
    name: httpd
    state: "{%raw%}{{ apache_service_state }}{%endraw%}"
    enabled: "{%raw%}{{ apache_service_enabled }}{%endraw%}"

- name: Ensure firewalld has selected state.
  firewalld:
    port: "{%raw%}{{ apache_listen_port }}/tcp{%endraw%}"
    permanent: true
    state: "{%raw%}{{ apache_firewall_state }}{%endraw%}"

- name: Configure Apache listen port.
  lineinfile:
    path: /etc/httpd/conf/httpd.conf
    regexp: '^Listen '
    insertafter: '^#Listen '
    line: "Listen {%raw%}{{ apache_listen_port }}{%endraw%}"
  notify: restart apache
```
Edit <c>handlers/main.yml</c>:
```yaml
# Handlers for ansible-role-apache
- name: restart apache
  service:
    name: httpd
    state: restarted
```

## The final test
Time for a final full test <c>molecule test -s kvm</c>:
```python
--> Validating schema /root/ansible-role-apache/molecule/docker/molecule.yml.
Validation completed successfully.
--> Validating schema /root/ansible-role-apache/molecule/kvm/molecule.yml.
Validation completed successfully.
--> Test matrix

└── kvm
    ├── lint
    ├── destroy
    ├── dependency
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    └── destroy

--> Scenario: 'kvm'
--> Action: 'lint'
--> Executing Yamllint on files found in /root/ansible-role-apache/...
Lint completed successfully.
--> Executing Flake8 on files found in /root/ansible-role-apache/molecule/kvm/../shared/tests/...
Lint completed successfully.
--> Executing Ansible Lint on /root/ansible-role-apache/molecule/shared/playbook.yml...
Lint completed successfully.
--> Scenario: 'kvm'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    ok: [localhost] => (item={'box': u'centos/7', 'name': u'instance1', 'groups': [u'rhel']})

    TASK [Populate instance config] ************************************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    skipping: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=2    changed=0    unreachable=0    failed=0


--> Scenario: 'kvm'
--> Action: 'dependency'
Skipping, missing the requirements file.
--> Scenario: 'kvm'
--> Action: 'syntax'

    playbook: /root/ansible-role-apache/molecule/shared/playbook.yml

--> Scenario: 'kvm'
--> Action: 'create'

    PLAY [Create] ******************************************************************

    TASK [Create molecule instance(s)] *********************************************
    changed: [localhost] => (item={'box': u'centos/7', 'name': u'instance1', 'groups': [u'rhel']})

    TASK [Populate instance config dict] *******************************************
    ok: [localhost] => (item={u'IdentityFile': u'/tmp/molecule/ansible-role-apache/kvm/.vagrant/machines/instance1/libvirt/private_key', '_ansible_parsed': True, u'changed': True, '_ansible_no_log': False, '_ansible_item_result': True, u'PasswordAuthentication': u'no', u'UserKnownHostsFile': u'/dev/null', '_ansible_item_label': {'box': u'centos/7', 'name': u'instance1', 'groups': [u'rhel']}, u'log': u'/tmp/molecule/ansible-role-apache/kvm/vagrant-instance1.out', 'item': {'box': u'centos/7', 'name': u'instance1', 'groups': [u'rhel']}, u'LogLevel': u'FATAL', u'HostName': u'192.168.121.225', u'IdentitiesOnly': u'yes', 'failed': False, u'Host': u'instance1', u'User': u'vagrant', u'invocation': {u'module_args': {u'config_options': {}, u'provider_raw_config_args': None, u'platform_box': u'centos/7', u'provider_override_args': None, u'instance_raw_config_args': None, u'provision': False, u'platform_box_url': None, u'provider_cpus': 2, u'state': u'up', u'instance_interfaces': [], u'instance_name': u'instance1', u'platform_box_version': None, u'provider_memory': 512, u'provider_options': {}, u'provider_name': u'libvirt', u'force_stop': False}}, u'StrictHostKeyChecking': u'no', u'Port': u'22', '_ansible_ignore_errors': None})

    TASK [Convert instance config dict to a list] **********************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=4    changed=2    unreachable=0    failed=0


--> Scenario: 'kvm'
--> Action: 'prepare'

    PLAY RECAP *********************************************************************

--> Scenario: 'kvm'
--> Action: 'converge'

    PLAY [converge] ****************************************************************

    TASK [Gathering Facts] *********************************************************
    ok: [instance1]

    TASK [ansible-role-apache : Install Apache.] ***********************************
    changed: [instance1]

    TASK [ansible-role-apache : Ensure Apache has selected state.] *****************
    changed: [instance1]

    TASK [ansible-role-apache : Ensure firewalld has selected state.] **************
    changed: [instance1]

    TASK [ansible-role-apache : Configure Apache listen port.] *********************
    ok: [instance1]

    PLAY RECAP *********************************************************************
    instance1                  : ok=5    changed=3    unreachable=0    failed=0


--> Scenario: 'kvm'
--> Action: 'idempotence'
Idempotence completed successfully.
--> Scenario: 'kvm'
--> Action: 'side_effect'
Skipping, side effect playbook not configured.
--> Scenario: 'kvm'
--> Action: 'verify'
--> Executing Testinfra tests found in /root/ansible-role-apache/molecule/kvm/../shared/tests/...
    ============================= test session starts ==============================
    platform linux2 -- Python 2.7.5, pytest-3.9.3, py-1.7.0, pluggy-0.8.1 -- /root/molecule_lab/bin/python2.7
    rootdir: /root/ansible-role-apache/molecule, inifile:
    plugins: testinfra-1.16.0
collected 3 items                                                              

    ../shared/tests/test_default.py::test_apache_is_installed[ansible:/instance1] PASSED [ 33%]
    ../shared/tests/test_default.py::test_apache_running_and_enabled[ansible:/instance1] PASSED [ 66%]
    ../shared/tests/test_default.py::test_port_80_is_listening[ansible:/instance1] PASSED [100%]

    =========================== 3 passed in 6.00 seconds ===========================
Verifier completed successfully.
--> Scenario: 'kvm'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Destroy molecule instance(s)] ********************************************
    changed: [localhost] => (item={'box': u'centos/7', 'name': u'instance1', 'groups': [u'rhel']})

    TASK [Populate instance config] ************************************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=3    changed=2    unreachable=0    failed=0
```

<notify_green><p markdown="1">As you have seen in this post Molecule can be run locally for development, but it can also run in Continuous Integration pipeline and that I will cover in my next blog post.</p></notify_green>


BR, pgustafs
