---
layout: post
title: Arch Linux setup in ansible
---

<p class="message">
The article describes ansible code, which can be found <a href="https://github.com/zpieslak/arch">here</a>
</p>

When searching for a reliable and fast system for everyday use, Arch Linux seems to be the best solution for me. It allows installing only the needed components, and its rolling release model brings updates almost instantly.

However, as always, there is a cost. Since Arch does not make any assumptions of what will be installed, almost every package needs to be configured manually. At one hand, it is extremely time-consuming and requires a lot of effort to just install the operating system. But on the other hand, we gain a knowledge about the system, its internals, and we are certain that we have absolute minimum on our local computer.

The other issue, that can arise, is how to store the knowledge. When, for example, some configuration was changed or some package needed to be installed. Obviously, for plain installation, the [Arch Wiki](https://wiki.archlinux.org/title/installation_guide) can be followed and for user configuration - our dotfiles can be copied. If additional work is needed, just any other manual steps can be performed, like installing a package or changing a line of a file.

The above solution could work for most cases, but for me, it was a little odd, to keep knowledge in multiple places and remember what manual steps needed to be performed. Initially I tried to write a bash script, but I thought that there should be a better solution, and maybe I do not need to reinvent the wheel.

During some research, I found Ansible, which seemed to be a better suited tool. Internally, it uses python and provides some kind of idempotency to the run command. It provides a nice messaging system (for example when command fails or succeed) and is also very well suited for performing server changes or deploys.

After moving all the commands to ansible setup, the process of recreating personal enviroment is very simple. As on the diagram below:

      +----------------------------+       +-----------------------------+
      |  Laptop with ansible code  |  ssh  |  New laptop with empty disk |
      |     (ansible client)       |------>|       (ansible server)      |
      +----------------------------+       +-----------------------------+

## Manual changes

Although most of the process is automated, in order to establish a ssh connection, some initial manual changes on both *ansible server* and *ansible client* sides are needed.

### Ansible server

1. Run Arch Linux live iso.
1. Sign in as root and connect to local LAN (here a wifi example is presented):
  * Check the name of the wifi interface

            ip link

    * Create wpa_supplicant configuration file (assuming the previous command returned `wlan0` as interface name).

            # cat /etc/wpa_supplicant/wpa_supplicant-wlan0.conf

            ctrl_interface=/var/run/wpa_supplicant
            ctrl_interface_group=wheel
            update_config=1
            network={
             ssid="my_ssid"
             psk="my_password"
            }

    * Restart wpa_supplicant service

            systemctl restart wpa_supplicant@wlan0


1. Note the local ip address to be able to verify the connection on *ansible client*.

        ip addr show wlan0

### Ansible client

1. Confirm there is ssh connection with *ansible server* (assuming the previous command returned `ansible_server_ip`)

        ssh root@ansible_server_ip

1. Copy the group variables configuration file (assuming our host is called `ansible_server`)

        cp group_vars/all group_vars/ansible_server

1. Adjust the configuration as needed.

1. Change `ansible_server_ip` in `hosts` file

        # cat hosts

        [ansible_server]
        ansible_server_ip

## Run ansible

1. Perform initial commands that partition disk and install bootable base.

        ansible-playbook -k site.yml -t disk,boot -l ansible_server

1. Remove livecd. Restart *ansible server* and wait for boot.

1. Run any other commands that are needed.

        ansible-playbook -k site.yml -t system,user,ag,git,vim -l ansible_server

System should be now usable. We can install any additional packages we need on the server (see `roles` directory).

## Conclusion

Although configuration can be kept just as dotfiles and in form of personal notes, I found myself much more confident, if I can recreate my local setup very quick, without need of writing notes (as notes becomes de-facto ansible code) and (what is even worse) not to solve the same problem multiple times.
