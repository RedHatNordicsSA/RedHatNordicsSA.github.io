---
layout: post
title:  "How to deploy Red Hat Enterprise Linux 8, really fast"
date:   2019-04-24 12:23:00 +0200
tags:
  - rhel8
  - composer
author: mglantz
---

<p><banner_h>Summary</banner_h></p>
This blogpost will be fairly straight forward. It will walk you through the
process of deploying Red Hat Enterprise Linux 8, really really fast. To do so,
we'll be using the new ''composer'' tooling in RHEL8 to generate a disk image
file you can deploy from, to speed things up considerably. To enable you go from
zero to logging into a new virtual machine instance of RHEL8, in seconds instead
of minutes.

On a KVM platform with modern hardware, we can do provisioning in below
one second and login to our new VM, in less than 7 seconds.

With that said, this post is about doing VM deployment much faster, but without
turning it into rocket science.

<p><banner_h>Why</banner_h></p>
Before we get started, a fair question is

```
My current VM deployment takes a couple of minutes.. is that not enough?
```

The short answer is, no. The long answer is, to cater for developers who wants
to be able to test their code in fresh environments, waiting 5 minutes for a VM
to spin up, is pain. Even worse, kickstarting a VM typically takes 30 minutes.

Also, to cater for future FaaS ([Function-as-a-Service](https://en.wikipedia.org/wiki/Function_as_a_service)) and other
get-it-when-you-need-it architectures where a VM get's provisioned or spun-up
triggered by incoming traffic to a load balancer or etc, then you need to get
much faster.

<p><banner_h>You convinced me, teach me how</banner_h></p>
Let's get started. From here on, there will only be step-by-step instructions.

# Summary of things needed
* A virtual machine to run your RHEL8 build server
* A VM platform to deploy virtual machines on


# Install Red Hat Enterprise Linux 8 on a virtual machine.
This VM will be your build server, where you build new RHEL8 installation images.
[Instructions for how to download and install RHEL8 are linked here.](https://developers.redhat.com/rhel8/install-rhel8/)

# On your RHEL8 VM, follow below instructions:

* Install composer and yum repo tools, used to build your installation image
of RHEL8:
  ```
  dnf install -y dnf-utils createrepo_c composer-cli lorax-composer httpd git
  ```

* If you do not have a SSH key which you want to manage your VM with, generate
one now. DO NOT set a passphrase.
  ```
  ssh-keygen
  ```

* Set SELinux to permissive mode (atm, composer does not work with SELinux in
enforcing mode):
  ```
  setenforce 0
  ```

* (If you want to make the SELinux change permanent, edit:
  /etc/selinux/config)

* Enable and start Apache web server, it will be used to serve yum repositories
used in the RHEL8 build.
  ```
  systemctl enable httpd
  systemctl start httpd
  ```

* Sync RHEL8 to your RHEL8 composer server:
  ```
  reposync -n -p /var/www/html --repoid rhel-8-for-x86_64-baseos-htb-rpms --downloadcomps --download-metadata
  reposync -n -p /var/www/html --repoid rhel-8-for-x86_64-appstreams-htb-rpms --downloadcomps --download-metadat
  ```

* Disable all repos from Red Hat Network on your server, the Composer tooling
 requires this.
  ```
  subscription-manager repos disable=*
  ```

* Create local repositories from which we'll build your RHEL8 installation image
  ```
  createrepo -v /var/www/html/rhel-8-for-x86_64-baseos-htb-rpms/ -g $(ls /var/www/html/rhel-8-for-x86_64-baseos-htb-rpms/repodata/*comps.xml)
  createrepo -v /var/www/html/rhel-8-for-x86_64-apptreams-htb-rpms/ -g $(ls /var/www/html/rhel-8-for-x86_64-appstreams-htb-rpms/repodata/*comps.xml)
  ```

* Create local repository files which points to your local repos which you've now created
  ```
  cat << EOF >/etc/yum.repos.d/rhel.repo
  [baseos-rpms]
  name=baseos-rpms
  baseurl=http://localhost/rhel-8-for-x86_64-baseos-htb-rpms
  enabled=1
  gpgcheck=1
  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release


  [appstream-rpms]
  name=appstream-rpms
  baseurl=http://localhost/rhel-8-for-x86_64-appstream-htb-rpms
  enabled=1
  gpgcheck=1
  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
  EOF
  ```

* Edit the Composer kickstart for your VM platform and input below
optimizations, they'll will reduce the boot time of your RHEL8 instance down to
less than 4 seconds (depends on hardware):

  * For VMware, edit: /usr/share/lorax/composer/vmdk.ks
  * For OpenStack, edit: /usr/share/lorax/composer/openstack.ks
  * For Amazon EC2, edit: /usr/share/lorax/composer/ami.ks
  * For general purpose KVM, edit: /usr/share/lorax/composer/qcow2.ks

  After the below lines
  ```
  rm /etc/machine-id
  touch /etc/machine-id
  ```

  And before the line
  ```
  %end
  ```

  Paste in below optimizations (feel free to add any other things you can think of):
  ```
  # Boot time optimization
  sed -i -e 's/GRUB_TIMEOUT=5/GRUB_TIMEOUT=0/' /etc/default/grub
  echo "GRUB_HIDDEN_TIMEOUT=0" >>/etc/default/grub
  echo "GRUB_HIDDEN_TIMEOUT_QUIET=true" >>/etc/default/grub
  sed -i -e 's/rhgb quiet/text quiet fastboot cryptomgr.notests tsc.reliable=1 rd.shell=0 ipv6.disable=1 rd.udev.log-priority=3 noprobe no_timer_check printk.time=1 usbcore.nousb cgroup_disable=memory/
  ' /etc/default/grub
  grub2-mkconfig -o /boot/grub2/grub.cfg

  for item in firewalld.service auditd.service rhsmcertd.service tuned.service spice-vdagentd.socket remote-fs.target dnf-makecache.service chronyd.service kdump.service sssd.service; do systemctl disable $item; done
  ```

* Create your Composer blueprint. Create the file: /var/lib/lorax/composer/blueprints/rhel8-base.toml
and paste in:
  ```
  name = "rhel8-base"
  description = "A general purpose rhel8 image"
  version = "0.6.0"

  [[packages]]
  name = "curl"
  version = "*"

  [[packages]]
  name = "file"
  version = "*"

  [[packages]]
  name = "git"
  version = "*"

  [[packages]]
  name = "gnupg2"
  version = "*"

  [[packages]]
  name = "sudo"
  version = "*"

  [[packages]]
  name = "tar"
  version = "*"

  [[packages]]
  name = "xz"
  version = "*"

  [[customizations.user]]
  name = "root"
  key = "ssh-rsa key-here"
  ```

* Ensure that you have replaced 'ssh-rsa key-here' with your public SSH key.
If you created one as instructed earlier inthe blog, which you want to use,
paste in everything outputted from below command:
  ```
  cat ~/.ssh/id_rsa.pub
  ```

* Push your new RHEL8 blueprint and ensure that it's published:
  ```
  composer-cli blueprints push rhel8-base.toml
  composer-cli blueprints list|grep rhel8-base
  ```

* Now it's time to build your RHEL8 image, depending on what VM platform you
are to deploy on, you use below commands:

  * For VMware:
    ```
    composer-cli compose start rhel8-base vmdk
    ```

  * For OpenStack:
    ```
    composer-cli compose start rhel8-base openstack
    ```

  * For Amazon:
    ```
    composer-cli compose start rhel8-base ami
    ```

  * For general purpose KVM:
    ```
    composer-cli compose start rhel8-base qcow2
    ```

  * Verify the build and when completed, download the disk image:
    ```
    composer-cli compose status
    composer-cli compose image UUID-OF-BUILD
    ```

# Deploying the virtual machine
If you are deploying on VMware, Amazon, OpenStack, make your disk image
available and deploy from it. How? You'll have to google it, this blog post does
not deal with that :-)

* If you are deploying the VM image on a KVM host, you can borrow some tooling
I've created.

* Clone the [https://github.com/mglantz/c_vm](c_vm) GitHub repository:
  ```
  git clone https://github.com/mglantz/c_vm
  ```

* Copy c_vm, provision_vm.yaml and template-vm.xml to the server you are deploying from and ensure you have ansible installed.

* Edit c_vm to reflect on where you have your image file, provision_vm.yaml and your template-vm.xml files.

* Run c_vm
  ```
  sudo c_vm name-of-vm
  ```

  Example output

    ```
    [mglantz@darkred templates]$ time sudo c_vm mtest15
    Formatting '/var/lib/libvirt/images/mtest15.qcow2', fmt=qcow2 size=16106127360 backing_file=/home/mglantz/code/ansible/vm/rhel8-base-0.6.0.qcow2 cluster_size=65536 lazy_refcounts=off refcount_bits=16
    Domain mtest15 created from /home/mglantz/code/ansible/vm/templates/mtest15.xml

    IP-ADDRESS OF SYSTEM: 192.168.122.181
     [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost
    does not match 'all'


    PLAY [Provision VM] *************************************************************************************

    TASK [Debug] ********************************************************************************************
    ok: [localhost] => {
        "msg": "IP of VM is: 192.168.122.181"
    }

    TASK [Add host into in-memory inventory] ****************************************************************
    changed: [localhost]

    TASK [Check so that server is online] *******************************************************************
    ok: [localhost]

    PLAY RECAP **********************************************************************************************
    localhost                  : ok=3    changed=1    unreachable=0    failed=0   


    real	0m6.663s
    user	0m1.041s
    sys	0m0.223s
    ```

* Enjoy your new fast deployment of RHEL8 :-)
