# Mobile Witness Self-Hosted Sample Scripts

If you don't wish to use the provided cloud hosts with Mobile Witness, the scripts here will give you a starting point to use a personal server as an upload location.

NOTE: Curl script to automate all these steps should probably be done to help people less technically inclined.

## Prerequisites

A LEMP stack (or similar) is required for hosting. This guide has been prepared with Debian Testing (11 July 16), Nginx 1.10.1 & PHP 7.0.8.

###### 1. OS Dependencies
Debian/Ubuntu APT: `sudo apt-get install nginx-full; sudo apt-get install php7.0-fpm`

###### 2. Nginx Web Server Config
/etc/nginx/sites-available/

```
server {
        listen 4321 ssl http2;              #Choose a port. Enable SSL to prevent MIIM. Remove http2 if nginx is old
        root /var/www/mobilewitness/html;   #Choose web file locations
        server_name SERVER_NAME;            #Server domain goes here
        
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;      #Nginx Auth file goes here
        
        access_log      /var/log/nginx/mobilewitness/access.log;
        error_log       /var/log/nginx/mobilewitness/error.log warn;

        client_max_body_size 50m;           #These directives control maximum file size upload allowed
        client_body_buffer_size 50m;        #These directives control maximum file size upload allowed

        server_tokens off;
        ssl_protocols TLSv1.2;
        ssl_certificate /etc/ssl/private/servercert.crt;      #SSL Certificate path
        ssl_certificate_key /etc/ssl/private/serverkey.key;   #SSL Key path
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;
        ssl_session_tickets on;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /etc/ssl/private/full_chain.pem;
        add_header Strict-Transport-Security "max-age=31536000; preload";

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php7.0-fpm.sock;
        }

}
```

###### 3. PHP Scripts
/var/www/mobilewitness

Folder structure for scripts is as follows:
```
+document root
|-html
 |-uploadFile.php       (controls where uploaded files are saved to)
 |-uploads              (these folders store uploaded files)
  |-Audio
  |-Image
  |-Video
  |-GPS
  |-Phone
```

uploadFile.php
```
<?php
        if($_SERVER['REQUEST_METHOD']=='POST'){
                $file_name = $_FILES['myFile']['name'];
                $file_size = $_FILES['myFile']['size'];
                $file_type = $_FILES['myFile']['type'];
                $temp_name = $_FILES['myFile']['tmp_name'];

                if ($file_type == 'audio/mp4'){
                        $location = "uploads/Audio/";
                } else if ($file_type == 'image/jpeg'){
                        $location = "uploads/Image/";
                } else if ($file_type == 'video/mp4'){
                        $location = "uploads/Video/";
                }else if ($file_type == 'text/plain'){
                        $location = "uploads/GPS/";
                } else if ($file_type == 'audio/AMR'){
                        $location = "uploads/Phone/";
                }

                move_uploaded_file($temp_name, $location.$file_name);
                echo "Uploaded";
        } else{
                echo "Error";
        }
?>
```
Note: You will have to set your /etc/php/7.1/fpm/php.ini to suitable values. E.g upload_max_filesize = 128M & post_max_size = 128M.
