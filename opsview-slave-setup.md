# Opsview slave install guide

## Prerequisites

### Disable SeLinux
To permanently disable SELinux, use your favorite text editor to open the file `/etc/sysconfig/selinux` as follows:
```
root@slave# vim /etc/sysconfig/selinux
```
![SELinux Enforcing Mode](https://www.tecmint.com/wp-content/uploads/2016/07/SELinux-Enforcing-Mode.png)

Then change the directive `SELinux=enforcing` to `SELinux=disabled` as shown in the below image.
```
SELINUX=disabled
```
![Disable SELinux Permanently](https://www.tecmint.com/wp-content/uploads/2016/07/Disable-SELinux.png)

Then, save and exit the file, for the changes to take effect, you need to reboot your system and then check the status of SELinux using sestatus command as shown:
```
root@slave# sestatus
```
![Check SELinux Status](https://www.tecmint.com/wp-content/uploads/2016/07/Check-SELinux-Status.png)

### Install the required packages.
```
root@slave# yum install -y epel-release vim
```
## Pre-install Tasks
These steps are to be performed on the *new slave server*, unless otherwise stated.
1. Create the nagios and nagcmd groups:
```
roo@slave# groupadd nagios 
root@slave# groupadd nagcmd
```
2. Create the nagios user and set its password to a known and secure value:
```
root@slave# useradd -g nagios -G nagcmd -d /var/log/nagios -m nagios 
root@slave# passwd nagios 
```
3. Ensure nagios user has root access for specific commands via sudo (note also the nagios user should have sudo on its PATH to use this correctly):
```
root@slave# visudo
```
add the following line
```
nagios ALL = NOPASSWD: /usr/local/nagios/bin/install_slave
```
4. On the *master server*, copy the nagios SSH public key from the master to the slave server:
```
root@master# su nagios 
nagios@master$ ssh-keygen -t dsa  # Creates your SSH public/private keys if they do not currently exist 
nagios@master$ ssh-copy-id -i .ssh/id_dsa.pub {slave_hostname} 
```
The .ssh directory should be mode 0700 and the id_dsa file should be 0600.
You should be able to connect to the slave server from the Opsview master server without passwords:
```
root@master$ su nagios 
nagios@master$ ssh {slave_hostname} # Should be logged into slave system
```
5. Set up the profile for the Nagios user on the *slave server*:
```
nagios@slave$ echo "test -f /usr/local/nagios/bin/profile && . /usr/local/nagios/bin/profile" >> ~/.profile
nagios@slave$ chown nagios:nagios ~/.profile
```
6. Copy the check_reqs and profile scripts from the master onto the slave as the nagios user
This should work without prompting for authentication:
```
nagios@master$ scp /usr/local/nagios/installer/check_reqs /usr/local/nagios/bin/profile {slave_hostname}:
```
On the slave as the nagios user, source the profile then run check_reqs:
```
nagios@slave$ . ./profile 
nagios@slave$ ./check_reqs slave 
```
Fix any dependency issues listed.

7. Create the temporary drop directory on the slave.
On the slave, create the temporary directory to put the transfer files:
```
nagios@slave$ su root
root@slave# mkdir -p /usr/local/nagios/tmp  
root@slave# chown nagios.nagios /usr/local/nagios /usr/local/nagios/tmp  
root@slave# chmod 775 /usr/local/nagios /usr/local/nagios/tmp 
```
8. Check SSH TCP Port Forwarding on the slave.
In order to communicate with the master server, port forwarding must be enabled in `/etc/ssh/sshd_config` on the slave server. Ensure that the following option is
set to yes (default is yes):
```
AllowTcpForwarding yes
```
Restart the SSH server if this is changed.
```
root@slave# service sshd restart
```

## Setup of the slave
1. Within the master web interface, ensure that the slave host is set up on the master server `Configuration -> Hosts -> Create New Host`

*Note:* The host used for the slaves must have at least 1 service associated with it, otherwise a configuration reload will fail.

2. Within the master web interface, add the slave host as a monitoring server `Advanced -> Monitoring Servers -> Create New Monitoring Server`

3. From the *master server*, send the necessary configuration files to the slave host:
```
root@slave# su nagios
nagios@slave$ /usr/local/nagios/bin/send2slaves -t {slave_hostname} # Test connection 
/usr/local/nagios/bin/send2slaves {slave_hostname}
```
The slave name is optional and can be used when multiple slaves have been defined. This will produce an error `Errors requiring manual intervention: 1` and will detail running the commands in the next step.

4. On the slave server run the setup program:
```
nagios@slave$ su root 
root@slave# cd /usr/local/nagios/tmp && ./install_slave
```
*Note:* You may get the error message:
```
Can't access neither configuration file 
/usr/local/nagios/nmis/bin/../conf/nmis.conf, nor /etc/nmis/nmis.conf
```
This can be ignored. This will be hidden from Opsview 3.9.1.
5. Within the master web interface, reload the Opsview configuration on the master server `erver -> Status And Reload -> Reload Configuration`


## Source

* https://www.tecmint.com/disable-selinux-temporarily-permanently-in-centos-rhel-fedora/
* https://web.archive.org/web/20101123183111/http://docs.opsview.org:80/doku.php?id=opsview-community:slavesetup

