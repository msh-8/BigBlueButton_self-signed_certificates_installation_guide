# BigBlueButton installation with self-signed instruction
# Overview

The instruction goal is to provide a flexible and as simple as possible configuration for BigBlueButton installation with self-signed certificate. 
It was designed to be used by professional system administrators or for development/testing/education purposes.

# Caveats

In the order to effectively use the project for your particular purposes you have to understand the principles of Linux container
and have a base knowledge about Nginx, Shell script and self-signed certificates.

# Hardware requirments

* Memory: 8 to 16 GB
* CPU: 4 to 8 cores

# OS requirments

* Ubuntu server 20.04
* Internet connection to pull the packages

 # Repositories requirments

 * APT repository:
   * deb https://mirror.iranserver.com/ubuntu focal main restricted universe multiverse
   * deb https://mirror.iranserver.com/ubuntu focal-updates main restricted universe multiverse
   * deb https://mirror.iranserver.com/ubuntu focal-security main restricted universe multiverse
   * deb https://mirror.iranserver.com/ubuntu focal-backports main restricted universe multiverse

 * Mongodb repository:
     * repo.mongodb.org \
   <b> note: It is restricted from IR </b>

* Bigbluebutton repo:
  * ubuntu.bigbluebutton.org

# Certificate requirements:

* Valid certificate requirements:
    * Public domain name
    * Public IP address
      
* Self signed certificate requirements:
    * Issue in the linux os

# Containers requirements:

* Docker engine should be installed
* Mirror registry should be configured to pull the images: \
  
  <b> Note: Run the following command in linux bash
  ```
  sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
			{
  			"insecure-registries" : ["https://docker.arvancloud.ir"],
  			"registry-mirrors": ["https://docker.arvancloud.ir"]
			}
			EOF'
			sudo systemctl restart docker
  ```

  # How to install:
    In this example we use "bbb.mci.local" as a private domain name and 192.168.122.11 as private static ip address.

  * Configure the hostname and host file:
 
    ```
    hostnamectl set-hostname bbb.mci.local
    echo "192.168.122.11  bbb.mci.local" >> /etc/hosts
    ```
    # issue a certificate:
    The instruction domenstrates the installation of bigbluebutton with self signed certificate
    ```
    mkdir bbb
    cd bbb
    openssl req -x509 -out bbb.mci.crt -keyout bbb.mci.key -newkey rsa:2048 -nodes -sha256  -new -subj "/C=GB/CN=bbb.mci.local" \
                  -addext "subjectAltName = DNS:bbb.mci.local, DNS:192.168.122.1" \
                  -addext "certificatePolicies = 1.2.3.4"
    ```
    openssl syntaxes:
    * --addext "subjectAltName = DNS:bbb.mci.local, DNS:192.168.122.1" => please replace your domain name and your private IP address of server.
    * CN=bbb.mci.local => replace your private domain name.
    * -out: replace name of your cert name.
    * -keyout: replace name of your key name. \
    <b> note: leave other parameters out as they are.</b> \
    
    trust the issued certificate in localhost:
    ```
    sudo chmod 644 bbb.mci.crt
    sudo chown root:root bbb.mci.crt
    sudo chmod 600 bbb.mci.key
    sudo chown root:ssl-cert bbb.mci.key
    sudo cp bbb.mci.crt /etc/ssl/certs/bbb.mci.crt
    sudo cp bbb.mci.key /etc/ssl/private/bbb.mci.key
    sudo update-ca-certificate
    ```
    # Install BigBlueButton:

    ```
    wget -qO- https://raw.githubusercontent.com/bigbluebutton/bbb-install/v2.7.x-release/bbb-install.sh
    chmod +x bbb-install.sh
    ./bbb-install.sh -d -v focal-270 -s bbb.mci.local -e mostafa.shoaei@gmail.com -g -x
    ```
    bbb-install.sh syntaxes:
    * -d: Skip SSL certificates request
    * -g: Install Greenlight version 3
    * -x: Use Let's Encrypt certbot with manual dns challenges \

    Note: when the installation complete whether successfully or failurely please change the nginx configuration as follows: \
    vi /etc/nginx/sites-available/bigbluebutton
    ```
    server {
    listen   80;
    listen [::]:80;
    #server_name  192.168.122.11;
    server_name  bbb.mci.local;

    access_log  /var/log/nginx/bigbluebutton.access.log;

    # This variable is used instead of $scheme by bigbluebutton nginx include
    # files, so $scheme can be overridden in reverse-proxy configurations.
    set $real_scheme $scheme;

    # BigBlueButton assets and static content.
    location / {
      root   /var/www/bigbluebutton-default/assets;
      index  index.html index.htm;
      expires 1m;
    }

    # Include specific rules for record and playback
    include /etc/bigbluebutton/nginx/*.nginx; # an overriding set of files, possibly present
    }
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name bbb.mci.local;
        ssl_certificate /etc/ssl/certs/bbb.mci.crt;
        ssl_certificate_key /etc/ssl/private/bbb.mci.key;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;
        #ssl_protocols TLSv1.2 TLSv1.3;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        #ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        #ssl_prefer_server_ciphers off;
        #ssl_dhparam /etc/nginx/ssl/ffdhe4096.pem;
        access_log /var/log/nginx/bigbluebutton.access.log;
        error_log /var/log/nginx/bigbluebutton.error.log;
        set $real_scheme "https";
      location / {
        root   /var/www/bigbluebutton-default/assets;
        index  index.html index.htm;
        try_files $uri @bbb-fe;
        expires 1m;
      }
	 # Handle RTMPT (RTMP Tunneling).  Forwards requests
	 # to Red5 on port 5080
      location ~ (/open/|/close/|/idle/|/send/|/fcs/) {
        #proxy_pass         http://127.0.0.1:7443;
        proxy_pass         http://bbb.mci.local:7443;
        proxy_redirect     off;
        #proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "Upgrade";
        proxy_set_header   Host $host;
        client_max_body_size       100m;
        client_body_buffer_size    128k;
        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_buffering            off;
        keepalive_requests         1000000000;
      }
	   # Handle desktop sharing tunneling.  Forwards
	   # requests to Red5 on port 5080.
     location /deskshare {
        proxy_pass         http://127.0.0.1:7443;
        proxy_redirect     default;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        client_max_body_size       10m;
        client_body_buffer_size    128k;
        proxy_connect_timeout      90;
        proxy_send_timeout         90;
        proxy_read_timeout         90;
        proxy_buffer_size          4k;
        proxy_buffers              4 32k;
        proxy_busy_buffers_size    64k;
        proxy_temp_file_write_size 64k;
        include    fastcgi_params;
     }
     # This variable is used instead of $scheme by bigbluebutton nginx include
     # files, so $scheme can be overridden in reverse-proxy configurations.
     #set $real_scheme $scheme;
     set $real_scheme "https";
     # Include specific rules for record and playback
     include /etc/bigbluebutton/nginx/*.nginx;
    }
    ```
    Restart the services:
    ```
    systemctl restart nginx
    bbb-conf --restart
    ```
    # Trust self-signedd certificate for greenlight container:
    ```
    docker cp bbb.mci.crt greenlight-v3:/usr/loca/share/ca-certificates/
    docker exec -t greenlight-v3 update-ca-certificates
    docker restart greenlight-v3
    ```
    # Create admin account in greenlight front end:
    ```
    docker exec -it greenlight-v3 bundle exec rake admin:create
    ```
    Note: After run above command you will be seen the something like the following:
    ```
    User account was created successfully!
      Name: Administrator
      Email: admin@example.com
      Password: Administrator1!
      Role: Administrator
    ```
    # Test the website:
    Open the https://bbb.mci.local in the browser and login as the above inforamtion.
