---
title:  "Rebuild an Ubuntu 18.04 Virtual Environment for Vagrant"
date:   2020-12-22 16:31:49 -0500
tags: ["github", "troubleshooting", "vagrant"]
draft: false
---

The CI for the [BlueBanquise][bb-github] project runs in GitHub Actions hosted
runners, in an Ubuntu 18.04 virtual machine. The project runs unit tests for
the Ansible roles with the [molecule][molecule] tool.

There is a recurring problem with the unit test of the role **core.time** where
the idempotence test occasionally fails in GitHub Actions. Despite multiple
attempts to reproduce the issue locally, the molecule test always succeed in my
lab system. Usually, a new run of the CI job allows to successfully pass the
unit test. No big deal.

However, with the recent introduction of [tox][tox] in [PR#398][bb-pr398], the
CI is now testing a combination of Python 2.7 and 3 with Ansible 2.8, 2.9 and
2.10, which triggers 6 instances of the molecule test in a single run. This
results in a low pass rate due to one or two random failures per run.

The time has come to troubleshoot the issue further. For this, I will build a
virtual machine as close as possible to the **ubuntu-18.04** image used in
GitHub Actions hosted runners. I will run this virtual machine in my lab system
(Debian 10 KVM hypervisor). Fortunately, the folks at GitHub share the files
used to build the images in the [Virtual Environments][gh-actions-virtualenv]
repository.  The images are built with the [Packer][packer] tool and the
**Azure ARM** builder. I will modify the Packer template to use the **Qemu**
builder instead, which will create an image usable with libvirt and Vagrant.

Convert the Packer template
---------------------------

The first step is to clone the repo.

```shell
$ git clone https://github.com/actions/virtual-environments.git
$ cd virtual-environments
```

The template file for Ubuntu 18.04 is *images/linux/ubuntu1804.json*. To be
able to build an image for the KVM hypervisor, I need to edit the template and
replace the **builders** section to use the **qemu** type. The project
[packer-qemu-templates](https://github.com/jakobadam/packer-qemu-templates)
provides good examples.


```json
  "builders": [
    {
      "type": "qemu",
      "boot_command": [
          "{{ user `boot_command_prefix` }}",
          "/install/vmlinuz noapic ",
          "file=/floppy/{{ user `preseed` }}  ",
          "debian-installer={{ user `locale` }} auto locale={{ user `locale` }} kbd-chooser/method=us ",
          "hostname={{ user `hostname` }} ",
          "fb=false  debconf/frontend=noninteractive ",
          "keyboard-configuration/modelcode=SKIP ",
          "keyboard-configuration/layout=USA ",
          "keyboard-configuration/variant=USA console-setup/ask_detect=false ",
          "passwd/user-fullname={{ user `ssh_username` }} ",
          "passwd/user-password={{ user `ssh_password` }} ",
          "passwd/user-password-again={{ user `ssh_password` }} ",
          "passwd/username={{ user `ssh_username` }} ",
          "initrd=/install/initrd.gz -- <enter>"
      ],
      "disk_size": "{{ user `disk_size` }}",
      "floppy_files": [
          "config/{{ user `preseed` }}"
      ],
      "headless": "{{ user `headless` }}",
      "iso_checksum": "{{ user `iso_checksum` }}",
      "iso_checksum_type": "{{ user `iso_checksum_type` }}",
      "iso_urls": [
          "{{ user `iso_url` }}"
      ],
      "output_directory": "/scratch/tmp/packer/output-{{ user `vm_name` }}",
      "shutdown_command": "echo  '{{ user `ssh_password` }}'|sudo -S shutdown -P now",
      "ssh_password": "{{ user `ssh_password` }}",
      "ssh_username": "{{ user `ssh_username` }}",
      "ssh_wait_timeout": "10000s",
      "vm_name": "{{ user `vm_name` }}",
      "cpus": "{{ user `cpus`}}",
      "memory": "{{ user `memory` }}",
      "qemuargs": [
          [ "-display", "none" ],
          [ "-machine", "accel=kvm" ]
      ]
    }
  ],
```

Add variables for the qemu builder. Adjust the number of CPUs and amount of
memory to optimize the build process.

```json
{
 "variables": {
  "vm_name": "ubuntu1804",
  "cpus": "4",
  "memory": "4096",
  "disk_size": "65536",
  "locale": "en_US.UTF-8",
  "iso_url": "http://cdimage.ubuntu.com/ubuntu/releases/18.04/release/ubuntu-18.04.5-server-amd64.iso",
  "iso_checksum": "8c5fc24894394035402f66f3824beb7234b757dd2b5531379cb310cedfdf0996",
  "iso_checksum_type": "sha256",
  "hostname": "vagrant",
  "ssh_username": "vagrant",
  "ssh_password": "vagrant",
  "install_vagrant_key": "true",
  "boot_command_prefix": "<esc><esc><enter><wait>",
  "preseed" : "preseed.cfg",
  "headless": "", 
[...]
```

The template includes the variable **github_feed_token** with a null value.
Remove this line to avoid a failure.

Import the preseed.cfg file:

```console
$ curl -o images/linux/config/preseed.cfg \
  https://raw.githubusercontent.com/jakobadam/packer-qemu-templates/master/ubuntu/http/preseed.cfg
```

To configure the image for Vagrant usage, import the vagrant provisioner
script:

```console
$ curl -o images/linux/scripts/base/vagrant.sh \
  https://raw.githubusercontent.com/jakobadam/packer-qemu-templates/master/ubuntu/scripts/vagrant.sh
```

This provisioner script will configure passwordless sudo which is required for
next steps, thus it is required to add it **at the beginning** of the
provisioners section:

```json
"provisioners": [
 {
  "environment_vars": [
   "INSTALL_VAGRANT_KEY={{user `install_vagrant_key`}}",
   "SSH_USERNAME={{user `ssh_username`}}",
   "SSH_PASSWORD={{user `ssh_password`}}"
  ],
  "execute_command": "echo '{{ user `ssh_password` }}' | {{.Vars}} sudo -E -S bash '{{.Path}}'",
  "scripts": [
   "scripts/base/vagrant.sh"
  ],
  "type": "shell"
 },
[...]
```

The network device name changes when the VM is started with Vagrant. Add the
script below **at the end** of the provisioners section (replace useless
waagent).  After reboot, Vagrant will be able to connect to the node (but
Packer won't be anymore).

```diff
--- a/images/linux/ubuntu1804.json
+++ b/images/linux/ubuntu1804.json
@@ -350,8 +327,7 @@
  {   
      "type": "shell",
      "inline": [
-         "sleep 30",
-         "/usr/sbin/waagent -force -deprovision+user && export HISTSIZE=0 && sync"
+         "sed -i -e 's/ens3/ens5/' /etc/netplan/01-netcfg.yaml"
      ],
      "execute_command": "sudo sh -c '{{ .Vars }} {{ .Path }}'"
  }
```

The file `/etc/waagent.conf` does not exist in the image, remove the *sed*
commands from *images/linux/scripts/installers/configure-environment.sh*:

```diff
--- a/images/linux/scripts/installers/configure-environment.sh
+++ b/images/linux/scripts/installers/configure-environment.sh
@@ -8,11 +8,6 @@ echo ImageOS=$IMAGE_OS | tee -a /etc/environment
 mkdir -p /etc/skel/.config/configstore
 echo 'export XDG_CONFIG_HOME=$HOME/.config' | tee -a /etc/skel/.bashrc

-# Change waagent entries to use /mnt for swapfile
-sed -i 's/ResourceDisk.Format=n/ResourceDisk.Format=y/g' /etc/waagent.conf
-sed -i 's/ResourceDisk.EnableSwap=n/ResourceDisk.EnableSwap=y/g' /etc/waagent.conf
-sed -i 's/ResourceDisk.SwapSizeMB=0/ResourceDisk.SwapSizeMB=4096/g' /etc/waagent.conf
-
 # Add localhost alias to ::1 IPv6
 sed -i 's/::1 ip6-localhost ip6-loopback/::1     localhost ip6-localhost ip6-loopback/g' /etc/hosts

```

Finally, add the **post-processors** section to convert the image to a Vagrant
box.

```json
"post-processors": [
    {   
        "keep_input_artifact": false,
        "output": "/scratch/tmp/box/{{.Provider}}/ubuntu1804-1.box",
        "type": "vagrant",
        "vagrantfile_template": "{{ user `vagrantfile_template` }}"
    }
]
```

Build the Ubuntu image for Vagrant and libvirt
----------------------------------------------

The template is now ready, I can build the image. During the build, Packer will
write intermediate files to /tmp/. Because the size of the image can be huge,
sometimes it is better to use a different temporary directory by setting
*TMPDIR*.

```shell
$ export TMPDIR=/scratch/tmp/packer
$ mkdir -p $TMPDIR
$ cd images/linux/
$ packer build ubuntu1804.json
```

{{< hint info >}}
**NOTE**\
To enable the debug, set PACKER_LOG=1.
{{< /hint >}}

The image size is (very) huge. In my environment, the packer command failed
with ENOSPACE despite using a file system with 124 GB of free space for the
build.

Reduce the list of tools to install
-----------------------------------

To reduce the image size, let's remove some useless (to me) tools:

 - {{template_dir}}/scripts/installers/aws.sh
 - {{template_dir}}/scripts/installers/swift.sh
 - {{template_dir}}/scripts/installers/codeql-bundle.sh
 - {{template_dir}}/scripts/installers/dotnetcore-sdk.sh
 - {{template_dir}}/scripts/installers/firefox.sh
 - {{template_dir}}/scripts/installers/google-chrome.sh
 - {{template_dir}}/scripts/installers/google-cloud-sdk.sh
 - {{template_dir}}/scripts/installers/leiningen.sh
 - {{template_dir}}/scripts/installers/mercurial.sh
 - {{template_dir}}/scripts/installers/mysql.sh
 - {{template_dir}}/scripts/installers/nvm.sh 
 - {{template_dir}}/scripts/installers/bazel.sh
 - {{template_dir}}/scripts/installers/oras-cli.sh
 - {{template_dir}}/scripts/installers/php.sh
 - {{template_dir}}/scripts/installers/postgresql.sh
 - {{template_dir}}/scripts/installers/vercel.sh
 - {{template_dir}}/scripts/installers/netlify.sh
 - {{template_dir}}/scripts/installers/android.sh
 - {{template_dir}}/scripts/installers/azpowershell.sh
 - {{template_dir}}/scripts/installers/hosted-tool-cache.sh
 - {{template_dir}}/scripts/installers/test-toolcache.sh

The SoftwareReport tools will gather some information about the software
installed in the image. For each tool removed previously, I need to remove
these from the report:

 - Get-NodeVersion
 - Get-SwiftVersion
 - Get-BazelVersion
 - Get-BazeliskVersion
 - Get-CodeQLBundleVersion
 - Get-GoogleCloudSDKVersion
 - Get-LeiningenVersion
 - Get-HGVersion
 - Get-NewmanVersion
 - Get-NvmVersion
 - Get-AWSCliVersion
 - Get-AWSCliSessionManagerPluginVersion
 - Get-AWSSAMVersion
 - Get-NetlifyCliVersion
 - Get-ORASCliVersion
 - Build-PHPTable
 - Get-ChromeVersion
 - Get-ChromeDriverVersion
 - Get-FirefoxVersion
 - Get-GeckodriverVersion
 - Get-DotNetCoreSdkVersions
 - Get-AzModuleVersions
 - Get-PostgreSqlVersion
 - Build-MySQLSection
 - Build-CachedToolsSection
 - Build-AndroidTable

The result file is still huge:

```console
$ ls -lh /scratch/tmp/box/libvirt/ubuntu1804-1.box 
-rw-r--r-- 1 bruno bruno 15G oct.  17 23:01 /scratch/tmp/box/libvirt/ubuntu1804-1.box
```

Add the new box to vagrant and init the Vagrantfile:

```console
$ export VAGRANT_HOME=/scratch/tmp/vagrant/
$ vagrant box add /scratch/tmp/box/libvirt/ubuntu1804-1.box --name gh-ubuntu1804
==> box: Box file was not detected as metadata. Adding it directly...
==> box: Adding box 'gh-ubuntu1804' (v0) for provider: 
    box: Unpacking necessary files from: file:///scratch/tmp/box/libvirt/ubuntu1804-1.box
==> box: Successfully added box 'gh-ubuntu1804' (v0) for 'libvirt'!
$ vagrant init gh-ubuntu1804
```

To use an alternate storage pool, add this configuration to the Vagrantfile:

```ruby
  config.vm.provider :libvirt do |domain|
    domain.storage_pool_name = "lab"
  end
```

Start the VM and connect via SSH to start the debug session:

```console
$ vagrant up
$ vagrant ssh
```

Cleanup
-------

Once the debug session is complete, destroy the VM and remove the image.

```console
$ vagrant destroy -f
==> default: Removing domain...

$ vagrant box remove gh-ubuntu1804
Removing box 'gh-ubuntu1804' (v0) with provider 'libvirt'...
Vagrant-libvirt plugin removed box only from your LOCAL ~/.vagrant/boxes directory
From Libvirt storage pool you have to delete image manually(virsh, virt-manager or by any other tool)

$ sudo virsh vol-delete gh-ubuntu1804_vagrant_box_image_0.img lab
Vol gh-ubuntu1804_vagrant_box_image_0.img deleted

$ rm /scratch/tmp/box/libvirt/ubuntu1804-1.box
```

Keep track in GitHub
--------------------

To keep track of these changes, I forked the upstream project in GitHub and
created a new branch [qemu_builder][gh-virtualenv-qemu_builder].

[bb-github]: https://github.com/bluebanquise/bluebanquise/
[bb-pr398]: https://github.com/bluebanquise/bluebanquise/pull/398
[molecule]: https://molecule.readthedocs.io/en/latest/
[tox]: https://pypi.org/project/tox/
[gh-actions-virtualenv]: https://github.com/actions/virtual-environments
[packer]: https://www.packer.io/
[gh-virtualenv-qemu_builder]: https://github.com/btravouillon/virtual-environments/tree/qemu_builder
