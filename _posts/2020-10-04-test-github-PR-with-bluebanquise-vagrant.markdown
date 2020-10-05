---
layout: post
title:  "Test GitHub PR with bluebanquise-vagrant"
date:   2020-10-04 18:38:16 -0400
categories: bluebanquise vagrant
---

The [bluebanquise](https://github.com/bluebanquise/bluebanquise/) CI is
somewhat limited when it comes to test client-server setup.

This post will demonstrate how to test a PR with the
[bluebanquise-vagrant][bb-vagrant] project. In this blog post, I want to create
a virtual environment to test the [PR #397][bb-pr397] which implements the
customization of the port of the rsyslog server. At the time of writing, the
head is [4605aeb][bb-4605aeb].

The first step is to clone the repository. I personally use temporary
directories for ephemeral environments: 

```shell
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

```shell
$ cd bluebanquise-vagrant/tiny-hpc-cluster/
$ cp vagrant.yml.example vagrant.yml
```

Edit the `vagrant.yml` file to match the local environment.

Set the `os_iso` parameter. The ISO file will be attached to the first
management virtual machine and the default bootstrap script will mount it in the
repositories directory.

```yaml
os_iso: "/scratch/isos/CentOS-8.2.2004-x86_64-dvd1.iso"
```

Define an alternate storage pool if requested:

```yaml
storage_pool_name: "lab"
```

Define a prefix for the virtual machines that match the PR. It helps to create
several instances with the same environment (e.g. one for development, the
other for testing).

```yaml
default_prefix: pr397
```

Save the configuration file and check `vagrant status`.

```shell
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

<pre>$ vagrant up management1
Bringing machine &apos;management1&apos; up with &apos;libvirt&apos; provider...
<font color="#C4A000"><b>==&gt; management1: You assigned a static IP ending in &quot;.1&quot; to this machine.</b></font>
<font color="#C4A000"><b>==&gt; management1: This is very often used by the router and can cause the</b></font>
<font color="#C4A000"><b>==&gt; management1: network to not work properly. If the network doesn&apos;t work</b></font>
<font color="#C4A000"><b>==&gt; management1: properly, try changing this IP.</b></font>
<font color="#C4A000"><b>==&gt; management1: You assigned a static IP ending in &quot;.1&quot; to this machine.</b></font>
<font color="#C4A000"><b>==&gt; management1: This is very often used by the router and can cause the</b></font>
<font color="#C4A000"><b>==&gt; management1: network to not work properly. If the network doesn&apos;t work</b></font>
<font color="#C4A000"><b>==&gt; management1: properly, try changing this IP.</b></font>
<b>==&gt; management1: Created volume larger than box defaults, will require manual resizing of</b>
<b>==&gt; management1: filesystems to utilize.</b>
<b>==&gt; management1: Creating image (snapshot of base box volume).</b>
<b>==&gt; management1: Creating domain with the following settings...</b>
<b>==&gt; management1:  -- Name:              pr397_management1</b>
[...]
</pre>
<pre>
$ vagrant up management2
Bringing machine &apos;management2&apos; up with &apos;libvirt&apos; provider...
<b>==&gt; management2: Created volume larger than box defaults, will require manual resizing of</b>
<b>==&gt; management2: filesystems to utilize.</b>
<b>==&gt; management2: Creating image (snapshot of base box volume).</b>
<b>==&gt; management2: Creating domain with the following settings...</b>
<b>==&gt; management2:  -- Name:              pr397_management2</b>
</pre>

The bootstrap script will clone the BlueBanquise repository in the first
management node and install some required packages. It will also setup the RPMS
repositories.

For CentOS, it will mount the CD-ROM (_os_iso_) to
`/var/www/html/repositories/centos/8/x86_64/os/`.

For BlueBanquise, it will download the RPMS from
<https://bluebanquise.com/repository/releases/1.3/el8/x86_64/bluebanquise/> to
`/var/www/html/repositories/centos/8/x86_64/bluebanquise/`.

Check the status.

```shell
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

```shell
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

From there, fetch `pull/ID/head:BRANCHNAME`.

```git
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

<pre>[bluebanquise]# git log --oneline master..
<font color="#C4A000">4605aeb (</font><font color="#06989A"><b>HEAD -&gt; </b></font><font color="#4E9A06"><b>pr397</b></font><font color="#C4A000">)</font> fix firewall and selinux to authorize customer ports
<font color="#C4A000">74bae4c</font> add log_client_rsyslog_server_port variable to customize rsyslogd port defaulted to UDP 514 to something else
<font color="#C4A000">96ecfb7</font> add log_client_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
<font color="#C4A000">48b4a35</font> add log_server_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
<font color="#C4A000">8998614</font> add log_server_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
<font color="#C4A000">2da49c4</font> add log_server_rsyslog_port variable to customize rsyslogd port defaulted to UDP 514 to something else
<font color="#C4A000">a80ed7a</font> Merge pull request #1 from bluebanquise/master</pre>

It's now time to test the Ansible role. First, setup the minimal management
configuration.

```shell
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

```shell
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

```shell
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

<pre>
[bluebanquise]# ansible-playbook playbooks/logs.yml --limit management2
[...]
TASK [log_client : Allow syslog port into SELinux] *****************************
Sunday 04 October 2020  20:53:04 -0400 (0:00:01.024)       0:00:04.104 ******** 
<font color="#CC0000">An exception occurred during task execution. To see the full traceback, use -vvv. The error was: ModuleNotFoundError: No module named &apos;seobject&apos;</font>
<font color="#CC0000">failed: [management2] (item=tcp) =&gt; changed=false </font>
<font color="#CC0000">  ansible_loop_var: item</font>
<font color="#CC0000">  item: tcp</font>
<font color="#CC0000">  msg: Failed to import the required Python library (policycoreutils-python) on management2&apos;s Python /usr/libexec/platform-python. Please read module documentation and install in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter</font>
<font color="#CC0000">An exception occurred during task execution. To see the full traceback, use -vvv. The error was: ModuleNotFoundError: No module named &apos;seobject&apos;</font>
<font color="#CC0000">failed: [management2] (item=udp) =&gt; changed=false </font>
<font color="#CC0000">  ansible_loop_var: item</font>
<font color="#CC0000">  item: udp</font>
<font color="#CC0000">  msg: Failed to import the required Python library (policycoreutils-python) on management2&apos;s Python /usr/libexec/platform-python. Please read module documentation and install in the appropriate location. If the required library is installed, but Ansible is using the wrong Python interpreter, please consult the documentation on ansible_python_interpreter</font>
</pre>

The role is failing with a Python error. Thanks to
[bluebanquise-vagrant][bb-vagrant], I am able to quickly test a code change and
provide feedback in the PR.

[bb-pr397]: https://github.com/bluebanquise/bluebanquise/pull/397
[bb-4605aeb]: https://github.com/bluebanquise/bluebanquise/pull/397/commits/4605aeb2efc7641e415305c1eeb328682cb7e49f
[bb-vagrant]: https://github.com/bluebanquise/bluebanquise-vagrant/
