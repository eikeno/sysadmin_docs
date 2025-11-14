[TOC]

# apache
## Name-Based Vhost Configuration

Hosts file for server and clients:

    192.168.1.80 hr.cloud.com sales.cloud.com

Install apache:

    dnf install -y httpd

/etc/httpd/conf.d/hr.conf:
```apacheconf
<Directory  /var/www/html/hr>
    Require all granted
    AllowOverride None
</Directory>

<VirtualHost 192.168.1.80:80>
    ServerAdmin root@hr.cloud.com
    DocumentRoot /var/www/html/hr
    ServerName hr.cloud.com
    ErrorLog "logs/hr_error_log"
    CustomLog "logs/hr_access_log"   combined
</VirtualHost>
```

/etc/httpd/conf.d/sales.conf:
```apacheconf
<Directory  /var/www/html/sales>

Require all granted

AllowOverride None
</Directory>

<VirtualHost 192.168.1.80:80>
    ServerAdmin root@sales.cloud.com
    DocumentRoot /var/www/html/sales
    ServerName sales.cloud.com
    ErrorLog "logs/sales_error_log"
    CustomLog "logs/sales_access_log"   combined
</VirtualHost>
```

    mkdir -p /var/www/html/hr /var/www/html/sales
    echo "Hello sales" > /var/www/html/sales/index.html
    echo "Hello HR" > /var/www/html/hr/index.html
    
    firewall-cmd --permanent --add-service=http
    firewall-cmd --reload
    systemctl enable --now httpd

## Configuring Private Directories

See also: [](https://httpd.apache.org/docs/2.4/fr/howto/auth.html)

Create a Directory 'secret' in the Document Root Path

    mkdir /var/www/html/secret
    echo '<h1>hello</h1>' > /var/www/html/secret/index.html

Create Password

    htpasswd -c /etc/httpd/htpasswd user1

In **/etc/httpd/conf/httpd.conf** add:
```apacheconf
<Directory  "/var/www/html/secret">
    AuthName "Protected Content"
    AuthType  Basic
    AuthUserFile  /etc/httpd/htpasswd
    Require valid-user
</Directory>
```



## Deploy Basic CGI Application

- CGI Script must be Owned by the apache user and group.
- It uses mod_cgi.so module in apache.
- httpd.conf contains **"ScriptAlias /cgi-bin/ "/var/www/cgi-bin/"** that will execute the CGI Scripts.
- Selinux must be adjusted with the context **httpd_sys_script_exec_t** for directory where VGI resides.

### Example deployment

    dnf install -y php

Change the Permissions of the **/var/www/cgi-bin/test.php** file:

    chmod +x test.php
    chown apache:apache test.php

Adjust SELinux:

    semanage fcontext -a -t httpd_sys_script_exec_t "/var/www/cgi-bin/(/.*)"?
    restorecon -Rv /var/www/cgi-bin/

Then restart the httpd Service.

## Group based authorization

Create several users with **htpasswd [password_file] [username]**

Create the Group file and add the users:

/etc/httpd/developers:
```
developers : user1  user2  user3
```

In httpd.conf:
```apacheconf
<Directory  "/var/www/html/secret">
    AuthType  Basic
    AuthName "Secret Content"
    AuthGroupFile  /etc/httpd/developers
    AuthUserFile /etc/httpd/htpasswd
    /etc/httpd/htpasswd
    Require  group  developers
</Directory>
```

Restart the Service:

    systemctl restart httpd


## Configuring TLS Security

Install required packages:

    dnf install -y mod_ssl crypto-utils

Generate certificate:

    genkey cloud.com

Copy the Path of the Certificate and Private Key and past it on the file */etc/httpd/conf.d/ssl.conf*:
```
/etc/pki/tls/private/cloud.com.key
/etc/pki/tls/certs/cloud.com.crt
```

Restart the Service and allow the *https* Service from the firewalld

### Configuration for a virtual host
```apacheconf
<VirtualHost *:443>
     SSLEngine On
     SSLCertificateFile /etc/pki/tls/certs/cloud.com.crt
     SSLCertificateKeyFile /etc/pki/tls/private/cloud.com.key

     ServerAdmin root@cloud.com
     ServerName www.cloud.com
     DocumentRoot /var/www/html/
     ErrorLog "logs/cloud.com_error_log"
     CustomLog "ogs/cloud.com_access_log" combined
</VirtualHost>
```

# See also
- [http://www.yann.com/fr/optimiser-la-configuration-dapache-20/05/2011.html](http://www.yann.com/fr/optimiser-la-configuration-dapache-20/05/2011.html)
