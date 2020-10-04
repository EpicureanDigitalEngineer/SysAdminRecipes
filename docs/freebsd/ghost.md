# Installing the `ghost` blogging platform under FreeBSD 12.x



## Problem Statement

This recipe provides instructions to install and maintain the [`ghost`](https://ghost.org/) blogging platform under FreeBSD which is not available as a separate package. There are no detailed installation instructions on the ghost site, but the app runs well and true under FreeBSD.

## Concepts

`ghost` runs on the following stack: `NodeJS`, `MySQL` (`NGINX`|`HAProxy`|`Apache`), `mail`.  The required modules can be found [here](https://ghost.org/docs/install/ubuntu/). This recipe should be ideally be executed in a FreeBSD jail to allow isolation of the services but may also be installed on a standard system. For this recipe we consider the current machine has the address `` blog.example.com`` and no prior installation of any of the required stack services. A mail client should be available and operating on the system (e.g. `ssmtp`) to allow `ghost` to communicate with the outside world. We don't treat here how to expose the service to the internet securely (e.g. behind a reverse proxy like `HAProxy` or `Nginx`).

## Install the required stack

``` bash
[root@ghost /]$ pkg install node10 npm-node10 mysql57-server
```

Check that packages are correctly installed:

``` bash
[root@ghost /]$ node --version
v10.16.3
[root@ghost /]$ npm --version
6.11.3
```

## Configure MySQL

Prepare a secure installation of MySQL:

``` bash
[root@ghost /]$ sysrc mysql_enable="YES"
mysql_enable:  -> YES
[root@ghost /]$ service mysql-server start
Starting mysql.
[root@ghost /]$ mysql_secure_installation
...
[root@ghost /]$ mysql -u root -p

root@localhost [(none)]> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'ENTERNEWPASSHERE';
root@localhost [(none)]> exit;
```
## Install ghost

### Create a system user for ghost

``` bash
[root@ghost /]$ adduser ghost
Username: ghost
Full name: Ghost User
Uid (Leave empty for default):
Login group [ghost]:
Login group is ghost. Invite ghost into other groups? []: www
Login class [default]:
Shell (sh csh tcsh bash rbash nologin) [sh]:
Home directory [/home/ghost]:
Home directory permissions (Leave empty for default):
Use password-based authentication? [yes]:
Use an empty password? (yes/no) [no]:
Use a random password? (yes/no) [no]: yes
Lock out the account after creation? [no]:
Username   : ghost
Password   : <random>
Full Name  : Ghost User
Uid        : 1001
Class      :
Groups     : ghost www
Home       : /home/ghost
Home Mode  :
Shell      : /bin/sh
Locked     : no
OK? (yes/no): yes
adduser: INFO: Successfully added (ghost) to the user database.
adduser: INFO: Password for (ghost) is: xyz
```

### Install ghost-cli

``` bash
[root@ghost /]$ npm i -g ghost-cli
...
[root@ghost /]$ mkdir -p /usr/local/www/ghost
[root@ghost /]$ chown ghost:ghost /usr/local/www/ghost
[root@ghost /]$ chmod 775 /usr/local/www/ghost
[root@ghost /]$ cd /usr/local/www/ghost
[root@ghost /usr/local/www/ghost]$ su ghost -c bash
[ghost@ghost /usr/local/www/ghost]$ ghost install
✔ Checking system Node.js version
✔ Checking current folder permissions
System checks failed with message: 'Operating system is not Linux'
Some features of Ghost-CLI may not work without additional configuration.
For local installs we recommend using `ghost install local` instead.
? Continue anyway? Yes
System stack check skipped
ℹ Checking operating system compatibility [skipped]
Local MySQL install not found. You can ignore this if you are using a remote MySQL host.
Alternatively you could:
a) install MySQL locally
b) run `ghost install --db=sqlite3` to use sqlite
c) run `ghost install local` to get a development install using sqlite3.
? Continue anyway? Yes
MySQL check skipped
ℹ Checking for a MySQL installation [skipped]
✔ Checking memory availability
✔ Checking for latest Ghost version
✔ Setting up install directory
✔ Downloading and installing Ghost v3.2.0
✔ Finishing install process
? Enter your blog URL: https://kd.blogs.dryllerakis.eu/
? Enter your MySQL hostname: localhost
? Enter your MySQL username: root
? Enter your MySQL password: [hidden]
? Enter your Ghost database name: ghost_prod
✔ Configuring Ghost
✔ Setting up instance
? Do you wish to set up "ghost" mysql user? Yes
✖ Setting up "ghost" mysql user
Nginx is not installed. Skipping Nginx setup.
ℹ Setting up Nginx [skipped]
Nginx setup task was skipped, skipping SSL setup
ℹ Setting up SSL [skipped]
? Do you wish to set up Systemd? No
ℹ Setting up Systemd [skipped]
Process manager 'systemd' will not run on this system, defaulting to 'local'
? Do you want to start Ghost? Yes
✔ Starting Ghost

Ghost uses direct mail by default. To set up an alternative email method read our docs at https://ghost.org/docs/concepts/config/#mail

------------------------------------------------------------------------------

Ghost was installed successfully! To complete setup of your publication, visit:

    https://blog.example.com/ghost/

```

### Create a start-up script for FreeBSD

Script: `/usr/local/etc/rc.d/ghost`

``` bash
#!/bin/sh

# PROVIDE: ghost
# REQUIRE: mysql-server
# KEYWORD: shutdown

PATH="/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin"

. /etc/rc.subr

name="ghost"
rcvar="ghost_enable"

load_rc_config ghost
: ${ghost_enable:="NO"}
: ${ghost_msg="Ghost did not start"}

start_cmd="${name}_start"
stop_cmd="${name}_stop"
stop_postcmd="echo Ghost stopped"
restart_cmd="${name}_restart"
required_dirs="$ghost_path"

ghost_start() {
    su -m ghost -c "cd $ghost_path && ghost start"
}

ghost_stop() {
    su -m ghost -c "cd $ghost_path && ghost stop"
}

ghost_restart() {
    su -m ghost -c "cd $ghost_path && ghost restart"
}


run_rc_command "$1"
​``` bash

and add it to the `rc.conf`:
​``` bash
[root@ghost /]$ sysrc ghost_enable="YES"
[root@ghost /]$ sysrc ghost_path="/usr/local/www/ghost"
```

Finally, update the config file to use `ssmtp` or `sendmail` for mailing:

``` bash
$ sed -ie 's/Direct/sendmail/g' config.production.json
```

### Configure ghost

  * Update configuration to use `sendmail` (`ssmtp`)
  * configure reverse proxy (`HAProxy`) to expose blog to the internet

## Updating ghost

  * ssh to the jail or OS
  * navigate to `/usr/local/www/ghost`
  * run `su ghost`
  * run `ghost update`
  * voilà
  * see [here](https://ghost.org/update/?v=2.0) for more info

## References

  - [[https://idontwatch.tv/installing-ghost-on-freebsd-11-1/]]
  - [[https://linoxide.com/linux-how-to/install-ghost-nginx-freebsd-10-2/]]