# Installing NGINX from Source

To install NGINX execute the following commands:

``` bash
cd ~/
apt-get install libpam-dev libpcre3-dev gcc make libssl-dev zlib1g-dev/stable git libxml2 libxml2-dev  libxslt1-dev -y
git clone https://github.com/vcmerritt/nginx-ldap-auth.git
git clone https://github.com/vcmerritt/nginx_http_auth_pam_module.git
git clone https://github.com/vcmerritt/nginx-websockify-module.git
git clone https://github.com/vcmerritt/nginx_devel_kit.git
git clone https://github.com/vcmerritt/nginx_cache_purge.git
git clone https://github.com/vcmerritt/nginx-dav-ext-module.git
wget http://nginx.org/download/nginx-1.17.9.tar.gz
tar -zxvf nginx-1.17.9.tar.gz
cd  nginx-1.17.9
./configure --prefix=/usr --conf-path=/etc/nginx/nginx.conf --modules-path=/etc/nginx/modules/ --add-dynamic-module=/root/nginx_http_auth_pam_module/ --with-http_v2_module --with-http_realip_module --with-http_gzip_static_module --with-http_auth_request_module --with-http_gunzip_module --with-http_ssl_module  --with-stream_ssl_module --with-threads --with-stream --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --with-http_secure_link_module --with-debug --add-dynamic-module=/root/nginx-websockify-module --add-dynamic-module=/root/nginx_devel_kit --add-module=/root/nginx_cache_purge --with-http_dav_module --add-module=/root/nginx-dav-ext-module
make
make install
mkdir /etc/nginx/sites-available
mkdir /etc/nginx/sites-enabled
```

## Create self-signed certificate
``` bash
mkdir /etc/nginx/ssl/
openssl req -new -x509 -days 3650 -nodes -out /etc/nginx/ssl/nextcloud.pem -keyout /etc/nginx/ssl/nextcloud.key
```

## Configure nginx for NextCloud and install INIT Scripts using GIT REPO
This process will make all changes identified in the following section to "Complete NGINX Config Changes", and also copies a new site to sites-enabled to enable NextCloud.  If you are not installing NextCloud, then remove the /etc/nginx/sites-enabled/nextcloud config file after running the commands below.

``` bash
cd ~/
git clone https://github.com/vcmerritt/nginx_nextcloud.git
cp nginx_nextcloud/nginx.service /lib/systemd/system/nginx.service
cp nginx_nextcloud/nginx.conf /etc/nginx/nginx.conf
cp nginx_nextcloud/sites-enabled/nextcloud /etc/nginx/sites-enabled
cp nginx_nextcloud/sites-available/nextcloud /etc/nginx/sites-available

#If you installed LibreOffice Online Server then copy this file to ensure that it works.
cp nginx_nextcloud/loolwsd.xml /etc/loolwsd

#Edit /etc/nginx/sites-available/nextcloud and set the Nextcloud server IP to the IP of the local system or 127.0.0.1
vi /etc/nginx/sites-available/nextcloud
upstream NextCloud {
  server 127.0.0.1:80;  ### CHANGE THIS LINE TO BE THE CORRECT IP!
}

#restart NGINX
systemctl restart nginx
systemctl enable nginx

#restart the LibreOffice Online Server (This must be done or you will not be able to edit documents).
systemctl restart loolwsd
```

<br>

# Force NextCloud SSL

``` bash
nano /var/www/html/nextcloud/config/config.php

#Add the following two lines to the file - careful to use the existing lines as examples so as not to break something.
'forcessl' => true,
'forceSSLforSubdomains' => true,

```

# Install and Configure Firewall
Install the firewall and configure it to only allow port 443 so that only https is permitted.   Port 80 and the other ports will be blocked after the configuration is performed, except for port 443, and port 22 (ssh) which is added by default.
``` bash
apt-get install iptables iptables-persistent -y
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state NEW -p udp -j ACCEPT
iptables -A INPUT -m state --state NEW -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -m state --state NEW -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
/sbin/service iptables save
```

<br>
-----------------------------   THIS IS ONLY FOR REFERENCE BELOW THIS LINE ----------------------------------------

<br>

**THE STEPS BELOW ARE ALREADY COMPLETED if you "Configured NGINX for NextCloud" above**


<br>

The changes listed below provide an overview of the changes made in the default configuration files that were copied as part of the previous steps performed.

<br>

#### Open /etc/nginx/nginx.conf and make the following changes
``` bash
  ##### change the pid entry to /run/nginx.pid
  vi  /etc/nginx/nginx.conf
  pid        /run/nginx.pid;

  ##### REMARK OUT THE DEFAULT Server Section for port 80


  ##### Add the line to load configs from sites-enabled
  add         include /etc/nginx/sites-enabled/*; to the end of the file
```

  ##### Create /lib/systemd/system/nginx.service and add the following contents
  ``` bash
  #NGINX INIT SCRIPT for /lib/systemd/system/nginx.service

# Stop  nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
 
#End of /lib/systemd/system/nginx.service


```
