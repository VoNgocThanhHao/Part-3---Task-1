# Install and config Wordpress

### Install **Wordpress, MySQL, php**:

    apt update 
>
    apt install nginx
>
    add-apt-repository ppa:ondrej/php
>
    apt install php7.4-fpm php7.4-curl php7.4-mysql php7.4-xml
>
    apt install mysql-server

### Config **MySQL**:

    mysql -h localhost -u root -P 3306 -p
>
    CREATE DATABASE wordpress; 
>
    CREATE USER wordpress@localhost IDENTIFIED BY '123456'; 
>
    GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER ON wordpress.* TO wordpress@localhost; 
>
    FLUSH PRIVILEGES; 
>
    quit 

### Config **Wordpress**:

    cd /var/www/ 
>
    curl -O https://wordpress.org/latest.tar.gz
>
    tar xzvf latest.tar.gz 
>
    cd wordpress 
>
    cp wp-config-sample.php wp-config.php 
>
    nano wp-config.php 

Change **db_name, db_user, db_password, db_host** to **wordpress, wordpress, 123456, localhost**:

    define( 'DB_NAME', 'wordpress' );
>
    define( 'DB_USER', 'wordpress' );
>
    define( 'DB_PASSWORD', '123456' );
>
    define( 'DB_HOST', 'localhost' );

Add `define('FS_METHOD','direct');` at the end.

Save and exit.

    chmod -R 777 ./* 
>
    nano /etc/nginx/sites-enabled/default

Change:

`root /var/www/html; `

`index index.html index.htm index.nginx-debian.html; `

`server_name _; `

To:

    root /var/www/wordpress;

    index index.html index.htm index.php; 

    server_name 0.0.0.0; 

Delete `#` at following line:

    location ~ \.php${ 

    include snippets/fastcgi-php.conf; 

    fastcgi_pass unix:/var/run/php7.4-fpm.sock; 

    }

Restart **nginx**:

    service nginx restart

# Config cache static file

    nano /etc/nginx/sites-available/default

Add following lines:

    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 365d;
    }

Like this:

    server{
        ...
        ...

        location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
            expires 365d;
        }
    }

# Install Amplify to NGINX Monitoring

Open your web browser with link https://amplify.nginx.com. Create account or login this website. You will get a API_KEY.

After that, log into your remote server to be monitored, via SSH and download the nginx amplify agent auto-install script using curl or wget command.

    wget https://github.com/nginxinc/nginx-amplify-agent/raw/master/packages/install.sh

OR

    curl -L -O https://github.com/nginxinc/nginx-amplify-agent/raw/master/packages/install.sh 

Now run the command below with superuser privileges using the sudo command, to install the amplify agent package (the API_KEY will probably be different, unique for every system that you add).

    sudo API_KEY='e126cf9a5c3b4f89498a4d7e1d7fdccf' sh ./install.sh 

*Note: You will possibly get an error indicating that sub_status has not been configured, this will be done in the next step.*

Create a new configuration file for stub_status under /etc/nginx/conf.d/:

    nano /etc/nginx/conf.d/sub_status.conf

Then copy and paste the following stub_status configuration in the file.

    server {
        listen 127.0.0.1:80;
        server_name 127.0.0.1;
        location /nginx_status {
            stub_status;
            allow 127.0.0.1;
            deny all;
        }
    }

Save and close the file.

Restart Nginx

    sudo systemctl restart nginx

![](https://i.imgur.com/YbcqGYI.png)

# Install OpenSSL Local 

    mkdir /etc/nginx/ssl
>
    cd /etc/nginx/ssl
>
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/key.key -out /etc/nginx/ssl/cert.crt
>
    chmod 777 ./
>
    nano /etc/nginx/sites-available/default

Add following lines:

    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    ssl_certificate /etc/nginx/ssl/cert.crt;
    ssl_certificate_key /etc/nginx/ssl/key.key;

Restart Nginx Service

    service nginx restart

![](https://i.imgur.com/A815btv.png)

# Config Nginx reverse proxy

    nano /etc/nginx/sites-available/default

Add following lines:

    server {
        listen 81;
        listen [::]:81;

        server_name _;

        set $ip_server 172.20.10.2;
        set $port_server 8080;

        location / {
                proxy_pass http://$ip_server:$port_server;
                try_files $uri $uri/ =404;
        }
    }

Restart Nginx Service

    service nginx restart

And now, when you access `172.20.206.101:81` you will redirect to `http://172.20.10.2:8080/`:

![](https://i.imgur.com/d4acyV3.png)

![](https://i.imgur.com/hQWXmlj.png)

# Block an IP or range IP

### Block 1 IP 

    nano /etc/nginx/sites-available/default

Add `deny <IP>` into the file:

    location / {
        ...
        deny    172.20.206.100;
        ...
    }

Like this:

![](https://i.imgur.com/VGm3pFQ.png)

Restart Nginx Service

    service nginx restart

Now IP `172.20.206.100` was blocked:

![](https://i.imgur.com/cGfDm7R.png)

### Block IP range

    nano /etc/nginx/sites-available/default

Add `deny <IP_range>` into the file:

    location / {
        ...
        deny    172.20.0.0/16;
        ...
    }

Restart Nginx Service

    service nginx restart

![](https://i.imgur.com/vabyH02.png)

# Use ApacheBench for web server performance testing

    apt-get update

    apt-get install -y apache2-utils

    ab <OPTIONS> <WEB_SERVER_ADDRESS>/<PATH>

`ab`’s options allow you to adjust the volume of requests, as well as (for specialized cases) their headers and request bodies. Some commonly used options include:

- `-n`: The number of requests to send

- `-t`: A duration in seconds after which ab will stop sending requests

- `-c`: The number of concurrent requests to make

Example:

    ab -n 100000 -c 1000 https://172.20.206.101/

I get result:

![](https://i.imgur.com/9UL6RYK.png)
