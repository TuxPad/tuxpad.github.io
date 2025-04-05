---
layout: post
title: Using KVM and VirtualBox on the same machine.
date: 2025-04-05
categories: [Linux, homelab]
tags: [kvm,virtualbox,virtualization,linux,troubleshooting]
toc: true
published: true
---

# Summary:
VirtualBox doesn't run while KVM modules are loaded. Here I will show you how you can get around this,
without rebooting or uninstalling either one, as well as how to automate it so that you can just launch the applications with their respective modules loaded.
# Intro:
As everyone who has an interest in IT knows, virtualization is pretty darn useful. Not only is it
the foundation of cloud computing, but it allows you to quickly spin up different VMs for development
and testing,or have a sandboxed environment for malware analysis, or a pre-configured box for
pentesting that can be discarded after the engagement.

It also fantastic for studying IT, as you can have a virtual lab environment, as you no longer have
to purchase a lot of expensive hardware. Building your own lab is great practice, and thera are a
bunch of labs that people have created for you to download to practice different techniques.

For hacking you can download labs from [VulnHub](https://www.vulnhub.com/),and [DVWA](https://github.com/digininja/DVWA),[Metasploitable 2](https://docs.rapid7.com/metasploit/metasploitable-2/)
& [3](https://github.com/rapid7/metasploitable3),[OWASP Juice Shop](https://owasp.org/www-project-juice-shop/),
and [GOAD](https://github.com/Orange-Cyberdefense/GOAD) are great as well.
# Scenario:
So I wanted to do the aforementioned GOAD, a lab consisting of five VMs.
Orange-Cyberdefense has made a python script for the setup, which can install it on different hypervisors,
which is very cool. While a proxmox server is on the wishlist, I have to install locally on my laptop, so I am
limited to using VirtualBox. I normally use [KVM](https://linux-kvm.org)/[QEMU](https://www.qemu.org/),
but when given such a cool lab for free, who am I to complain?


# Problem:
After installing and trying to run a vm in VirtualBox I was greeted with the error message:
```
Failed to open a session for the virtual machine ...
VirtualBox can't operate in VMX root mode. Please disable the KVM
kernel extension, recompile your kernel and reboot
(VERR_VMX_IN_VMX_ROOT_MODE).
```

Not cool. But as it turns out, there's no need to recompile, or to reboot for that matter.

As it turns out, VirtualBox doesn't play well with KVM and won't run if the KVM
modules are loaded. KVM, on the other hand, has no problem running with the VirtualBox modules
loaded.

# Solution:
In Linux we can remove  or insert kernel modules using `modprobe`or `rmmod`/`insmod`, and when installing
VirtualBox you even get a script, `vboxreload`, to load the modules (at least on Arch).

first we can check the kvm modules using `lmod | grep kvm`, which shows two modules loaded:
`kvm` and `kvm_intel` (since I have an Intel CPU).

if we unload the modules using either `sudo modprobe --remove kvm_intel kvm` or `sudo rmmod kvm_intel kvm`,
and then load the VirtualBox modules with `sudo vboxreload`.
If you don't have the shellscript then you can use modprobe to load vboxdrv, vboxnetadp & vboxnetflt.

Now you'll find that VirtualBox will run. If you then want to use KVM you'll have to load the modules again. You can do this with either modprobe or insmod, but note that while modprobe will look in
/usr/lib/modules, with insmod you have to specify the file yourself.

The module will be located in a
directory named after the current version of your kernel, so to avoid having to update the script every
time you update the kernel I recommend you use \`uname -r\` (note the backticks) in the pathname:
```
/sbin/insmod /lib/modules/`uname -r`/kernel/arch/x86/kvm/kvm.ko.zst
```

# Automation:
Having to do this manually every time seem a bit tedious though, so we can automate it.
If you launch the applications from the terminal then you could do one script for each application, which
insert/remove modules and launch the app, and make aliases for them and you're good to go.

I, however, prefer to use wofi, so I have to do a couple of extra steps.
Since I'm not launching from a terminal I need to add my scripts to the sudoers file, but since I
want to launch the applications from my user and I don't want to give a general NOPASSWORD for loading
kernel modules I split the scipts into a loading and a launching script.

vboxLoad.sh:
```
#!/bin/bash
/sbin/rmmod kvm_intel kvm
/usr/bin/vboxreload
```

vboxLaunch.sh:
```
#!/bin/bash
sudo /home/tuxpad/scripts/vbLoad.sh
/usr/bin/virtualbox
```

kvmLoad.sh:
```
modprobe --remove vboxnetadp vboxnetflt vboxdrv
modprobe kvm_intel kvm
```

kvmLaunch.sh:
```
sudo /home/tuxpad/scripts/kvmLoad.sh
/usr/bin/virt-manager
```
Then I added the scripts to the sudoers file:
`tuxpad ALL=(root) NOPASSWD: /home/tuxpad/scripts/vbLoad.sh,/home/tuxpad/scripts/kvmLoad.sh`

And finally I edited /usr/share/applications/virtualbox.desktop (and virt-manager.desktop) so
that wofi uses the the scripts to launch the applications.

Now I can Launch both VirtualBox and Virt-manager from wofi and they'll have the appropriate modules
loaded when I do so.
