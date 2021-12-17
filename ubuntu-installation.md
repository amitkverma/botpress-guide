# Deploying on ubuntu server

These steps shows how to deploy botpress on ubuntu server

## Prerequisite
* We recommend Ubuntu Server `18.04 LTS` or greater.
* Server must have open ports `3000` or you can customized it.
* Minimum size of the server must be 2 Core and 2GB RAM
* if you are planing to host it behind `NGINX` then make sure the requested ports are open (`80` and `443` for most of the cases).

## Installation steps

Download botpress file from source and run it.

```
sudo apt update
sudo apt install unzip

# Download the latest Linux binary
wget https://s3.amazonaws.com/botpress-binaries/botpress-$VERSION-linux-x64.zip

# Unzip the content in the current directory
unzip botpress-$VERSION-linux-x64.zip

# cd to botpress directory

# Launch the app
./bp
```
This will get you up and running locally on port 3000. If you encounter any error do check if ports are open or you have right installation version.

### Running botpress in background
install `PM2` if you already have then skip this step
```
sudo apt install npm
sudo npm install -g pm2
sudo pm2 startup
```
Once `PM2` is installed we will add botpress process in pm2
```
pm2 start './bp start -p'  

pm2 list        # check if botpress is added
```


#### Hosting botpress behind NGINX( Optional )
if you prefer hosting botpress behind a proxy server use the following method

Install nginx server
```
sudo apt install nginx
sudo mkdir /tmp/nginx_cache
```
Planing on hosting botpress on `HTTPS` instead of `HTTP`. Use following step otherwise skip it if you already have `SSL` certificates.
```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
sudo certbot --nginx
```

Once it's install open the file `/etc/nginx/sites-available/default` and add following lines in it.
```
http {
  # Disable sending the server identification
  server_tokens off;

  # Prevent displaying Botpress in an iframe (clickjacking protection)
  add_header X-Frame-Options SAMEORIGIN;

  # Prevent browsers from detecting the mimetype if not sent by the server.
  add_header X-Content-Type-Options nosniff;

  # Force enable the XSS filter for the website, in case it was disabled manually
  add_header X-XSS-Protection "1; mode=block";

  # Configure the cache for static assets
  proxy_cache_path /sr/nginx_cache levels=1:2 keys_zone=my_cache:10m max_size=10g
                inactive=60m use_temp_path=off;

  # Set the max file size for uploads (make sure it is larger than the configured media size in botpress.config.json)
  client_max_body_size 10M;

  # Configure access
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

  access_log  logs/access.log  main;
  error_log  logs/error.log;

  # Redirect unsecure requests to the HTTPS endpoint
  server {
    listen 80 default;
    server_name  localhost;

    return 301 https://$server_name$request_uri;
  }

  server {
    listen 443 http2 ssl;
    server_name localhost;

    ssl_certificate      cert.pem;
    ssl_certificate_key  cert.key;

    # Force the use of secure protocols only
    ssl_prefer_server_ciphers on;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

    # Enable session cache for added performances
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # Added security with HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";

    # Enable caching of assets by NGINX to reduce load on the server
    location ~ .*/assets/.* {
      proxy_cache my_cache;
      proxy_ignore_headers Cache-Control;
      proxy_hide_header Cache-Control;
      proxy_hide_header Pragma;
      proxy_pass http://localhost:3000;
      proxy_cache_valid any 30m;
      proxy_set_header Cache-Control max-age=30;
      add_header Cache-Control max-age=30;
    }

    # We need to add specific headers so the websockets can be set up through the reverse proxy
    location /socket.io/ {
      proxy_pass http://localhost:3000/socket.io/;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "Upgrade";
    }

    # All other requests should be directed to the server
    location / {
      proxy_pass http://localhost:3000;
    }
  }
}
```

Now the botpress should be available to you at `HTTPS`.
