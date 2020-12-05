---
title:  "Test BlueBanquise PR with Vagrant"
date:   2020-12-05 13:15:16 -0400
tags: ["bluebanquise", "vagrant"]
draft: false
---

The [BlueBanquise](https://github.com/bluebanquise/bluebanquise/) CI is
somewhat limited when it comes to test client-server setup.

This post will demonstrate how to test a pull request (PR) for BlueBanquise in
virtual machines with [Vagrant][vagrant] and the
[bluebanquise-vagrant][bb-vagrant] project. I want to create a virtual
environment to test the [PR #397][bb-pr397] which implements the customization
of the port of the rsyslog server. At the time of writing, the head of this PR
is [4605aeb][bb-4605aeb].

The hypervisor used in this blog post is a Debian 10 operating system with
KVM/libvirt.

The first step is to clone the repository. I prefer to use temporary
directories for ephemeral environments: 

```console
$ cd $(mktemp -d)
$ git clone https://github.com/bluebanquise/bluebanquise-vagrant.git
Cloning into 'bluebanquise-vagrant'...
remote: Enumerating objects: 390, done.
remote: Counting objects: 100% (390/390), done.
remote: Compressing objects: 100% (231/231), done.
remote: Total 390 (delta 174), reused 335 (delta 119), pack-reused 0
Receiving objects: 100% (390/390), 50.10 KiB | 1.86 MiB/s, done.
Resolving deltas: 100% (174/174), done.
```

To test a PR for BlueBanquise, I use the `tiny-hpc-cluster` environment. Enter
the directory and configure the environment in the `vagrant.yml` file.

```console
$ cd bluebanquise-vagrant/tiny-hpc-cluster/
$ cp vagrant.yml.example vagrant.yml
```

Edit the `vagrant.yml` file to match the local environment.

Set the `os_iso` parameter. The ISO file will be attached to the first
management virtual machine and the default bootstrap script will mount it in
the repositories directory
*/var/www/html/repositories/${DISTRIBUTION}/${MAJOR_RELEASE}/${ARCH}/os/* to
allow installation of packages with yum or dnf.

```yaml
os_iso: "/scratch/isos/CentOS-8.2.2004-x86_64-dvd1.iso"
```

Define an alternate storage pool, if requested:

```yaml
storage_pool_name: "lab"
```

Define a prefix for the virtual machines that match the PR. It helps to create
several instances with the same environment (e.g. one for development, another
for testing).

```yaml
default_prefix: pr397
```

Save the configuration file and check `vagrant status`.

```console
$ vagrant status
Current machine states:

management1               not created (libvirt)
management2               not created (libvirt)
c001                      not created (libvirt)
c002                      not created (libvirt)
c003                      not created (libvirt)
c004                      not created (libvirt)
login1                    not created (libvirt)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

Now, start the management nodes.

```console
$ vagrant up management1
Bringing machine 'management1' up with 'libvirt' provider...
==> management1: You assigned a static IP ending in ".1" to this machine.
==> management1: This is very often used by the router and can cause the
==> management1: network to not work properly. If the network doesn't work
==> management1: properly, try changing this IP.
==> management1: You assigned a static IP ending in ".1" to this machine.
==> management1: This is very often used by the router and can cause the
==> management1: network to not work properly. If the network doesn't work
==> management1: properly, try changing this IP.
==> management1: Created volume larger than box defaults, will require manual resizing of
==> management1: filesystems to utilize.
==> management1: Creating image (snapshot of base box volume).
==> management1: Creating domain with the following settings...
==> management1:  -- Name:              pr397_management1
[...]
```

```console
$ vagrant up management2
Bringing machine 'management2' up with 'libvirt' provider...
==> management2: Created volume larger than box defaults, will require manual resizing of
==> management2: filesystems to utilize.
==> management2: Creating image (snapshot of base box volume).
==> management2: Creating domain with the following settings...
==> management2:  -- Name:              pr397_management2
[...]
```

The default bootstrap script will clone the BlueBanquise repository in the
first management node and install some required packages. It will also setup
the RPMS repositories.

For CentOS, it will mount the CD-ROM (_os_iso_) to
`/var/www/html/repositories/centos/8/x86_64/os/`.

For BlueBanquise, it will download the RPMS from
<https://bluebanquise.com/repository/releases/1.3/el8/x86_64/bluebanquise/> to
`/var/www/html/repositories/centos/8/x86_64/bluebanquise/`.

Check the status.

```console
$ vagrant status
Current machine states:

management1               running (libvirt)
management2               running (libvirt)
c001                      not created (libvirt)
c002                      not created (libvirt)
c003                      not created (libvirt)
c004                      not created (libvirt)
login1                    not created (libvirt)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

Once the management nodes are up, it is possible to connect to the first
management node through SSH and enter the bluebanquise directory.

```console
$ vagrant ssh
==> management1: You assigned a static IP ending in ".1" to this machine.
==> management1: This is very often used by the router and can cause the
==> management1: network to not work properly. If the network doesn't work
==> management1: properly, try changing this IP.
==> management1: You assigned a static IP ending in ".1" to this machine.
==> management1: This is very often used by the router and can cause the
==> management1: network to not work properly. If the network doesn't work
==> management1: properly, try changing this IP.
[vagrant@management1 ~]$ sudo su -
[root@management1 ~]# cd /etc/bluebanquise/
[bluebanquise]#
```

From there, fetch the branch of the pull request (`pull/ID/head:BRANCHNAME`).

```console
[bluebanquise]# git fetch origin pull/397/head:pr397
remote: Enumerating objects: 35, done.
remote: Counting objects: 100% (35/35), done.
remote: Compressing objects: 100% (3/3), done.
remote: Total 61 (delta 30), reused 35 (delta 30), pack-reused 26
Unpacking objects: 100% (61/61), done.
From https://github.com/bluebanquise/bluebanquise
 * [new ref]         refs/pull/397/head -> pr397
[bluebanquise]# git checkout pr397 
Switched to branch 'pr397'
```

Check the list of commits since **master**:

```console
[bluebanquise]# git log --oneline master..
4605aeb (HEAD -> pr397 fix firewall and selinux to authorize customer ports
74bae4c add log_client_rsyslog_server_port variable to customize rsyslogd port defaulted to UDP 514 to something else
96ecfb7 add log_client_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
48b4a35 add log_server_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
8998614 add log_server_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
2da49c4 add log_server_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
a80ed7a Merge pull request #1 from bluebanquise/master
```

It's now time to test the Ansible role. First, setup the minimal management
configuration.

```console
[bluebanquise]# ssh-keyscan management1 > ~/.ssh/known_hosts
# management1:22 SSH-2.0-OpenSSH_8.0
# management1:22 SSH-2.0-OpenSSH_8.0
# management1:22 SSH-2.0-OpenSSH_8.0
[bluebanquise]# ansible-playbook playbooks/managements.yml --limit management1

PLAY [managements playbook] ***************************************************
[...]
PLAY RECAP *********************************************************************
management1: ok=77 changed=50 unreachable=0 failed=0 skipped=11 rescued=0 ignored=0   

Sunday 04 October 2020  23:32:53 +0000 (0:00:08.510)       0:02:21.152 ******** 
=============================================================================== 
pxe_stack -------------------------------------------------------------- 60.29s
advanced_dns_server ---------------------------------------------------- 20.82s
repositories_server ---------------------------------------------------- 11.24s
dhcp_server ------------------------------------------------------------ 11.01s
package ----------------------------------------------------------------- 8.51s
bluebanquise ------------------------------------------------------------ 6.70s
nfs_server -------------------------------------------------------------- 4.21s
firewall ---------------------------------------------------------------- 4.11s
time -------------------------------------------------------------------- 3.87s
repositories_client ----------------------------------------------------- 2.61s
gather_facts ------------------------------------------------------------ 2.47s
nic --------------------------------------------------------------------- 1.86s
set_hostname ------------------------------------------------------------ 1.11s
ssh_master -------------------------------------------------------------- 0.81s
hosts_file -------------------------------------------------------------- 0.74s
dns_client -------------------------------------------------------------- 0.72s
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 
total ----------------------------------------------------------------- 141.09s
```

Now, let's test the roles `log_server` and `log_client` which are modified in
[PR #397][bb-pr397]. There is an existing playbook in the default profile that
will apply these roles.

```yaml
[bluebanquise]# cat playbooks/logs.yml 
---
- name: log server playbook
  hosts: "management1"
  roles:
    - role: log_server
      tags: log_server

- name: log clients playbook
  hosts: "mg_computes,mg_logins,management2"
  roles:
    - role: log_client
      tags: log_client
```

Run the playbook on _management1_ first:

```console
[bluebanquise]# ansible-playbook playbooks/logs.yml --limit management1
[...]
PLAY RECAP ********************************************************************
management1: ok=10 changed=6 unreachable=0 failed=0 skipped=2 rescued=0 ignored=0   

Sunday 04 October 2020  19:38:27 -0400 (0:00:00.392)       0:00:13.275 ******** 
=============================================================================== 
log_server ------------------------------------------------------------- 11.76s
gather_facts ------------------------------------------------------------ 1.47s
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ 
total ------------------------------------------------------------------ 13.23s
```

Run the playbook on _management2_. This node will act as a client:

```console
[bluebanquise]# ansible-playbook playbooks/logs.yml --limit management2
[...]
TASK [log_client : Allow syslog port into SELinux] *****************************
Sunday 04 October 2020  20:53:04 -0400 (0:00:01.024)       0:00:04.104 ******** 
An exception occurred during task execution. To see the full traceback, use -vvv.
The error was: ModuleNotFoundError: No module named 'seobject'
failed: [management2] (item=tcp) => changed=false
  ansible_loop_var: item
  item: tcp
  msg: Failed to import the required Python library (policycoreutils-python) on
       management2's Python /usr/libexec/platform-python. Please read module
       documentation and install in the appropriate location. If the required
       library is installed, but Ansible is using the wrong Python interpreter,
       please consult the documentation on ansible_python_interpreter
An exception occurred during task execution. To see the full traceback, use -vvv.
The error was: ModuleNotFoundError: No module named 'seobject'
failed: [management2] (item=udp) => changed=false
  ansible_loop_var: item
  item: udp
  msg: Failed to import the required Python library (policycoreutils-python) on
       management2's Python /usr/libexec/platform-python. Please read module
       documentation and install in the appropriate location. If the required
       library is installed, but Ansible is using the wrong Python interpreter,
       please consult the documentation on ansible_python_interpreter
```

The role is failing with a Python error: *ModuleNotFoundError: No module named
'seobject'* .

Thanks to [bluebanquise-vagrant][bb-vagrant], I am able to quickly test a code
change and [provide feedback in the PR][bb-feedback-pr397].

[bb-pr397]: https://github.com/bluebanquise/bluebanquise/pull/397
[bb-4605aeb]: https://github.com/bluebanquise/bluebanquise/pull/397/commits/4605aeb2efc7641e415305c1eeb328682cb7e49f
[bb-vagrant]: https://github.com/bluebanquise/bluebanquise-vagrant/
[bb-feedback-pr397]: https://github.com/bluebanquise/bluebanquise/pull/397#discussion_r499321484
[vagrant]: https://www.vagrantup.com/
