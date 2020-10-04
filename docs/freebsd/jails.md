How to install (a set of) FreeBSD Jails (on 12.x)
========================================

## Problem Statement

Having installed a FreeBSD server, you now want to run multiple services (e.g. a [blogging platform](ghost.md), a NextCloud instance, ...) in a controlled and secure manner. Under Linux, you would have used `containers`. Under FreeBSD, you use [`jails`](https://www.freebsd.org/doc/handbook/jails.html). Jails appeared much earlier than containers and they addressed specifically the issue of isolating a set of processes from each other to obtain multi-tenancy of services in a more controllable and secure way. Various meta-systems exist to manage jails, [ezjail](https://www.freebsd.org/doc/handbook/jails-ezjail.html) among them.

This recipe explains how to create a set of jails in an already existing and configured FreeBSD 12.x server (see for example [here](secure-server) how to install a FreeBSD secure server in the cloud). It does not use any meta-tools like `ezjail` but proposes a set of conventions and concepts that facilitate the management of jails the low-level way. Therefore, it is addressed to those want to have a good understanding and control of what is happening under the hood.

Concepts
-----------

Running services in jails in FreeBSD is the "equivalent" to running services in containers in Linux (without the automation!). Jails can be configured in different ways to serve different purposes. Here, we consider that a jail is a miniature FreeBSD machine which is created to run specific services in a walled environment and which has its own network stack bridging the jail interfaces with the physical, uplink interface. We expected that the host FreeBSD server will eventually have multiple jails.

The configuration of the jail system resides in `/etc/jails`. It is in this file that we will be establishing a set of conventions that should make the management of the collection of jails (and the creation of new us) considerably more manageable. The file allows some parametrisation as well as the execution of scripts that automate a number of tasks like networking.

Conventions/patterns used include the following:

1. **Jail Location**: Jails are installed under under `/jail/<jail name>`. In case you are using `zfs` you should be creating a new dataset accordingly and take advantage of the `zfs` functionality to arrange for snapshots,  backups or migrations/cloning. 
2. **Jail privileges**: Jails are run as root - take good care to ensure the level of security you want to obtain from the system.
3. **Jail networking**: a `bridge0` is created (if not existing) upon start of the first jail . Each jail is assigned an [`epair`](https://www.freebsd.org/cgi/man.cgi?query=epair) interface named  `jail0` inside the jail. The outside part of the pair is added to the bridge. The uplink interface is also added to the bridge, effectively allowing jails to have a full networking stack (including loopback). When a jail is stopped, the corresponding `epair` interface is removed from the bridge and destroyed. 

Based on these conventions once your `/etc/jails` template has been prepared, adding a jail to the host system requires just two steps:

1. create a directory (or `zfs dataset`) and install the required jail files; perform initial basic configuration
2. append information about your jail to the `/etc/jails.conf` file and start the jail

The size of FreeBSD allows the jails to be relatively compact: 587MB for the base install or 1.6GB with the ports tree and current `portsnap(8)` information. As the jail is seen as an independent system, it will need to be maintained as a separate system (i.e. through `freebsd-update` and `pkg upgrade`).

 For the sake of example, the name of the host will be `host` while the the name of the jail to be created would be `myjail1`. 

## Installation process for a new jail

1. Configure and start the jail service with the conventions decided (if not done)
2. Create a space for your new jail and perform the basic installation/configuration
3. Update the `/etc/jails.conf` file to make your jail known
4. Start the jail and finalise the configuration

## Configure the Jail subsystem on the host

`/etc/rc.conf` and `/etc/jail.conf`

Add an entry for `myjail` on `/etc/hosts` (\'\'10.166.167.20 dc03\'\') of the host machine. If not already done, add `jail_enable="YES"`
to `/etc/rc.conf`

``` bash
################################################################
##      /etc/jail.conf                                         #
##      Basic Jail configuration for multiple Jails            #
##      The Epicurean Digital Engineer - 2020                  #
################################################################

## Global section

exec.system_user  = "root";
exec.jail_user    = "root";
mount.devfs;
allow.raw_sockets;
allow.sysvipc;
devfs_ruleset     = "5";

## Networking

$uplinkdev        = "re0";
vnet;
vnet.interface    = "jail0";  # default vnet interface
exec.prestart     = "ifconfig bridge0 > /dev/null 2> /dev/null || ( ifconfig bridge0 create up && ifconfig bridge0 addm $uplinkdev )";
exec.prestart    += "ifconfig $epair create up                 || echo 'Skipped creating epair(exists?)'";
exec.prestart    += "ifconfig bridge0 addm ${epair}a           || echo 'Skipped adding bridge member (already member?)'";
exec.created      = "ifconfig ${epair}b name jail0             || echo 'Skipped renaming ifdev to jail0 (looks bad...)'";
exec.clean;
exec.start        = "/bin/sh /etc/rc";
exec.stop         = "/bin/sh /etc/rc.shutdown";
exec.poststop     = "ifconfig bridge0 deletem ${epair}a";
exec.poststop    += "ifconfig ${epair}a destroy";

#
#	Information about individual jails
#	provided here for reference
#   to be added only when a jail is added!
#

myjail1 {
  host.hostname   = myjail1.example.com;
  path            = /jail/myjail1;
  $epair          = "epair0";  # must be unique in every jail
}

myjail2 {
  host.hostname   = myjail2.example.com;
  path            = /jail/myjail2;
  $epair          = "epair1";  # must be unique in every jail
}

myjail3 {
  host.hostname   = myjail2.example.com;
  path            = /jail/myjail3;
  $epair          = "epair2";  # must be unique in every jail
}

```



Create the jail
---------------

The complete Jail after finished installation takes less size then 587 MB. With complete FreeBSD Ports tree and current `portsnap(8)` information it takes about 1.6 GB of space.

``` bash
$ mkdir -p /jail/myjail # for a ufs installation
$ fetch -o - http://ftp.freebsd.org/pub/FreeBSD/releases/amd64/12.1-RELEASE/base.txz | tar --unlink -xpJf - -C /jail/myjail
```



Configure the host
------------------



Configure the jail
------------------

``` bash
################################################################
#  /etc/rc.conf for myjail - a jailed service		           #
################################################################
#
#
# Network Configuration
#
hostname="myjail.example.com"
defaultrouter="10.6.6.1"
ifconfig_jail0="inet 10.6.6.2 netmask 255.255.255.0"

# DAEMONS | yes
syslogd_flags="-s -s"

# DAEMONS | no
sshd_enable=NO
sendmail_enable=NONE
sendmail_submit_enable=NO
sendmail_outbound_enable=NO
sendmail_msp_queue_enable=NO

# OTHER
  clear_tmp_enable=YES
  clear_tmp_X=YES
  dumpdev=NO
  update_motd=NO
  keyrate=fast
```

``` conf
domain example.com
search example.com
nameserver 8.8.8.8
nameserver 1.1.1.1
```

``` bash
[root@host /jail]$ chroot /jail/myjail tzsetup
[root@host /jail]$ chroot /jail/myjail newaliases -v
```

Start the jail
--------------

Now that the jail has been created, you may start it and check
everything works.

```bash
[root@host /jail]$ service jail start myjail1
Starting jails: myjail1.
[root@host /jail]$ jexec myjail1 csh
```

## Upgrade jail to the latest patch level

To upgrade your new jail to the latest patch level use `freebsd-update -b` as follows:

```bash
[root@host /jail]$ freebsd-update -v /jail/myjail1 fetch
[root@host /jail]$ freebsd-update -v /jail/myjail1 install
```

!!! Note
    This presupposes that you are already on the latest patch level for your host system. If not, perform the `freebsd-update` on the host first to ensure that your kernel and userland versions will match (use `uname -UK` to verify if in doubt).

### Install the pkg system (latest version)

To allow to receive quicker updates, we switch to the \"latest version\"
pkg repository.

``` shell-session
root@myjail1:/ $ sed -i '' s/quarterly/latest/g /etc/pkg/FreeBSD.conf
root@myjail1:/ $ pkg update
The package management tool is not yet installed on your system.
Do you want to fetch and install it now? [y/N]: y
Bootstrapping pkg from pkg+http://pkg.FreeBSD.org/FreeBSD:12:amd64/latest, please wait...
Verifying signature with trusted certificate pkg.freebsd.org.2013102301... done
[myjail1.example.com] Installing pkg-1.12.0...
[myjail1.example.com] Extracting pkg-1.12.0: 100%
Updating FreeBSD repository catalogue...
[myjail1.example.com] Fetching meta.txz: 100%    944 B   0.9kB/s    00:01
[myjail1.example.com] Fetching packagesite.txz: 100%    6 MiB   1.6MB/s    00:04
Processing entries: 100%
FreeBSD repository update completed. 31945 packages processed.
All repositories are up to date.
root@dc03:/ $ pkg install -y bash

```

### Install packages you need

Now, install any packages you may need. I tend to install my favourite
shell `bash`