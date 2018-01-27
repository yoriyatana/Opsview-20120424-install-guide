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
nagios@slave$ echo "test -f /usr/local/nagios/bin/profile && . /usr/local/nagios/bin/profile" >> ~nagios/.profile
nagios@slave$ chown nagios:nagios ~nagios/.profile
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

Using The Slave
To make use of the slave host, amend each host configuration and set the Monitored By value to the correct server

Configuration->Hosts->{Edit Host}->Monitored By->{slave_host}
Alternatively, you can drag and drop the host between servers on the Monitoring servers list page.

Slave Web Interface
The opsview slaves can have a standard nagios web interface, but this is not enabled by default. To enable it:

1. Add the apache web server user (i.e. the user that apache runs as) to the nagcmd group, in order to read the htpasswd file. On Debian this is normally www-data - please check your system.

sudo usermod -G nagcmd www-data
2. Create an Apache configuration file by:

sudo su -
cp /usr/local/nagios/installer/apache_opsview_slave.conf /etc/apache2/sites-available/opsview_slave
vi /etc/apache2/sites-available/opsview_slave
Tailor as necessary for your site (if required) and then enable the new configuration (the default configuration should not be needed)

sudo a2ensite opsview_slave
sudo a2dissite default
3. stop and start apache (not restart as the new group membership needs to be activated)

sudo /etc/init.d/apache2 stop
sudo /etc/init.d/apache2 start
You should now be able to access the web pages on the slave using the same login credentials as the Opsview master server.

If you have any issues with accessing the web interface, check the Apache logs for error messages.

Using Slaves with reversed SSH tunnels
The reverse ssh tunnels are useful if security policy only allow slaves to initiate conversations to the master. A tunnel is started from slave to master, and then the master is able to start new communications with slave as required.

On master, edit opsview.conf and set:

$slave_initiated = 1;
$slave_base_port = 25800;
Restart opsview and opsview-web.

The master is now ready for reverse connections.

You will need to access slave in some way for the initial install. Create the nagios groups and user and exchange SSH keys: the slave needs the public key of the master and the master needs public key of slave.

Create the slave on the monitoring servers page. On the list page, hover over the slave node. This should then tell you the slave port number.

On the slave firstly install the autossh program/package, and then as nagios user run:

ssh -N -R {slave_port_number}:127.0.0.1:22 opsviewmaster
This process will not exit on slave.

On the master run the following as the nagios user:

/usr/local/nagios/bin/dosh -i -s {slavename} uname -a
This commands checks the connectivity to slaves.

Install Opsview with send2slaves {slave_name}. If this is a new slave you will need to manually intervene as root - check instructions onscreen. If this is a pre-exsting slave, rerun send2slaves {slave-name} again.

When installed, on the slave, create /usr/local/nagios/etc/opsview-slave.conf:

MASTER="master.opsview.org"
SLAVE_PORT={slave_port_number}
Test from the slave with:

/etc/init.d/opsview-slave test
/etc/init.d/opsview-slave start
/etc/init.d/opsview-slave status
On master, use dosh uname -a to test connectivity to all slaves.

All communications with master should now work correctly and a reload can now be performed.


## Source

* https://www.tecmint.com/disable-selinux-temporarily-permanently-in-centos-rhel-fedora/
* https://web.archive.org/web/20101123183111/http://docs.opsview.org:80/doku.php?id=opsview-community:slavesetup

