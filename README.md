# Opview-20120424-install-guide

##Prerequisites

* Disable SeLinux
To permanently disable SELinux, use your favorite text editor to open the file `/etc/sysconfig/selinux` as follows:
```
#vim /etc/sysconfig/selinux
```
SELinux Enforcing Mode
SELinux Enforcing Mode

Then change the directive `SELinux=enforcing` to `SELinux=disabled` as shown in the below image.
```
SELINUX=disabled
```
Disable SELinux Permanently
Disable SELinux Permanently

Then, save and exit the file, for the changes to take effect, you need to reboot your system and then check the status of SELinux using sestatus command as shown:
```
$ sestatus
```
Check SELinux Status
Check SELinux Status

* Packages
Instal the required packages. The opsview meta package should bring in most packages however there are a couple that aren't declared dependencies.
```
# yum install -y mysql-server java rrdtool-perl redhat-lsb perl-Net-SNMP perl-MOP
```

##Configuration

###MySQL
Add the following to the [ mysqld] section of `'/etc/my.cnf'`:
```
bind-address = localhost

innodb_file_per_table=1
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=256M
```
 Start the MySql server:
```
# chkconfig mysqld on
# service mysqld start
```
Change the MySql root password:
```
# /usr/bin/mysqladmin -u root password password
```
###Nagios user setup
Add the following line to the nagios users `'.bash_profile'`.
```
nagios$ echo "test -f /usr/local/nagios/bin/profile && . /usr/local/nagios/bin/profile" >> ~/.bash_profile
```
###Create the Nagios database
```
# /usr/local/nagios/bin/db_mysql -u root -ppassword
```
###OpsView
As the nagios user edit the file `'/usr/local/nagios/etc/opsview.conf'` and populate it with passwords.
```
# /usr/local/nagios/bin/db_opsview db_install
# /usr/local/nagios/bin/db_runtime db_install
# /usr/local/nagios/bin/db_odw db_install
# /usr/local/nagios/bin/db_reports db_install

# /usr/local/nagios/bin/rc.opsview gen_config
```
Due to a known limitation add the following to the file `' /usr/local/opsview-web/opsview_web_local.yml'`. This is a new file that didn't exist in the distribution:
```
Controller::Root:
  authtkt_ignoreip: 1
```
###Start the web service.
```
# service opsview-web start
```
###Initial setup
Use a web browser to view the web interface on port 3000 of the host. The initial credentials are:
```
Username	admin
Password	initial
 ```

Screensnap 2011-09-29_174514.png
 

 

Nginx Web Server
Instead of the stock recommended Apache httpd configuration use Nginx as a web server to server the static content and proxy the dynamic pages.  This is based on the information in the Apache httpd sample configuration '/usr/local/nagios/installer/apache_proxy.conf
'.

server {
    listen       [::]:80;
    server_name  _;

    location ~* ^/(error_pages/|javascript/|stylesheets/|help/|images/|xml/|favicon.ico|graphs/|static/|media/) {
       expires                 7d;
       add_header              Cache-Control public;
       access_log              off;
       root                    /usr/local/nagios/share;
    }
    location /static/nmis/ {
       expires                 7d;
       add_header              Cache-Control public;
       access_log              off;
       root                    /usr/local/nagios/nmis/htdocs;
    }
    location / {
        proxy_pass             http://localhost:3000;
    }
}
 

Nginx Reverse proxy
Use Nginx as a front-door reverse proxy with SSL offload. Note that the Catalyst engine requires the 'X-Forwarded-...' headers to be set so that the URL's are rewritten correctly.

server {
    listen               [::]:80;
    server_name          opsview.lucidsolutions.co.nz;
    location / {
        # redirect to secure page [permanent | redirect]
        rewrite ^ https://opsview.lucidsolutions.co.nz$request_uri? permanent;
    }
}

server {
    listen               [::]:443 ssl;
    server_name          opsview.lucidsolutions.co.nz;
    keepalive_timeout    70;

    ssl                  on;
    ssl_certificate      certs/opsview.lucidsolutions.co.nz.startssl.crt;
    ssl_certificate_key  certs/opsview.lucidsolutions.co.nz.key;
    ssl_ciphers          ALL;

    include /etc/nginx/semitrusted-networks;

    access_log  /var/log/nginx/opsview.lucidsolutions.co.nz.access.log  main;

    location / {
        proxy_pass              http://10.20.11.3:80;
        proxy_set_header        Host             $host;
        proxy_set_header        X-Real-IP        $remote_addr;
        proxy_set_header        X-Forwarded-For  $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Host $http_host;
        proxy_set_header        X-Forwarded-Port 443;
    }
}
Residuals
No caching of web site content
Static content is served by the OpsView perl module
