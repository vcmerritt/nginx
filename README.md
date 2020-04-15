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
#Edit the .htaccess file in /var/www/html/nextcloud and add the lines below right under Rewrite On
nano /var/www/html/nextcloud/config/config.php    # Add the following lines right after the datadirectory path.
  
  'overwrite.cli.url' => 'https://YourIPorFullServerName',       
  'overwriteprotocol' => 'https',

#Create a default index.html file that redirects to NextCloud via SSL
#MAKE SURE THAT YOU CHANGE THE IPorHOSTNAMEofNextCloudServer to the correct IP
cat <<EOF > /var/www/html/index.html
<script>
window.location = "https://IPorHostNameofNextCloudServer/nextcloud/";   
</script>
EOF

```

# Install and Configure Firewall
Install the firewall and configure it to only allow port 443 so that only https is permitted.   Port 80 and the other ports will be blocked after the configuration is performed, except for port 443, and port 22 (ssh) which is added by default.
``` bash
apt-get install iptables iptables-persistent -y
export PATH=$PATH:/usr/sbin
iptables -F
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp -j DROP
iptables-save > /etc/iptables/rules.v4
```

## Test the NextCloud Server Installation
To test the server installation and ensure you are using SSL, open a web browser and navigate to:

``` HTML
https://IPorHostnameofNextCloudServer
or
https://IPorHostnameofNextCloudServer/nextcloud/         #Make sure you add  the trailing /
```

<br>

## Add Collabora App in NextCloud
Navigate to Apps, and add the Collabora Online Server component to nextcloud.  After you have added the extension to NextCloud, then go to settings for the Collabora server, and configure the URL to: <br>
<br>

https://IPorHostnameofNextCloudServer

<br>
Note:  Don't forget to select the checkbox immediately below the URL in settings to trust certificates if you are using a self-signed certificate.

## Add Collabora Online App to Nextcloud
1)  Login to NextCloud as the nextadmin account
2)  Select Apps, and Add the Collabora Online application
3)  Select Settings --> Collabora Online and enter https://IPorNameofNextCloudServer
4)  Go to folders, create a new document and test the Collabora Online functionality.

## Configure LDAPS Integration for NextCloud

### From within NextCloud configure the following:
You must create the __svc_nextcloud__ user accont and the (__OU=Users,OU=MYHQ,DC=mydomain,DC=com__), (__OU=Groups,OU=MYHQ,DC=mydomain,DC=com__), (__OU=Admins,OU=MYHQ,DC=mydomain,DC=com__) Organizational Units prior to completing these steps.

1)  Login to NextCloud as the nextadmin account <br>
2)  Select Apps, and choose Disabled Apps <br>
3)  Enable and Download LDAP / AD Integration  <br>
4)  Select Settings -->  LDAP / AD Integration <br>
5)  Enter the following: <br>
     Server:   ``` ldaps://sambadc01.mydomain.com   Port:  636 ```  <br>
     USER DN:  ``` CN=svc_nextcloud,OU=Users,OU=MYHQ,DC=mydomain,DC=com ```  <br>
     PASSWORD:  ``` Password for user ``` <br>
     One Base DN Per Line:  <br>
     ```
                OU=Users,OU=MYHQ,DC=mydomain,DC=com
                OU=Admins,OU=MYHQ,DC=mydomain,DC=com
     ``` 
6)  With __Windows Domain Controllers__ - Select the Login Attributes Tab, and edit the LDAP Filter to add the following: <br>
     ```     (&(|(objectclass=person))(|(samaccountname=%uid)(|(mailPrimaryAddress=%uid)(mail=%uid))(|(memberOf=%uid)))) ``` 
     
7)  With __SAMBA Domain Controllers__ - Select the Login Attributes Tab, and edit the LDAP Filter to add the following: <br>
     ```     (&(|(objectclass=person))(|(samaccountname=%uid)(|(mailPrimaryAddress=%uid)(mail=%uid))(|(primaryGroupID=%uid)))) ```

8)  Select Groups Tab, and edit the LDAP Filter to add the following: <br>
    ```  bash
    (&(|(objectclass=organizationalPerson)(objectclass=organizationalUnit)(objectclass=person)(objectclass=top)(objectclass=user))(|(cn=Next_*))) 
    ```  
     
  
     __Note:__  Using the filter in step 8 above, only AD groups that begin with Next_ will appear as valid groups in NextCloud.
<br>
<br>
