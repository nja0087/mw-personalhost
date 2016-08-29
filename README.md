# Mobile Witness Self-Hosted Sample Scripts

If you don't wish to use the provided cloud hosts with Mobile Witness, the scripts here will give you a starting point to use a personal server as an upload location.

NOTE: Curl script to automate all these steps should probably be done to help people less technically inclined.

## Prerequisites

A LEMP stack (or similar) is required for hosting. This guide has been prepared with Debian Testing (11 July 16), Nginx 1.10.1, MariaDB 10.1.15, & PHP 7.0.8.

###### 1. OS Dependencies
Debian/Ubuntu APT: `sudo apt-get install nginx-full;  sudo apt-get install mariadb-server; sudo apt-get install php7.0-fpm; sudo apt-get install php7.0-mysql;`

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

###### 3. MariaDB (MySQL) Statements

```
CREATE TABLE `recordings` (
  `timestamp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `latitude` decimal(10,6) NOT NULL,
  `longitude` decimal(10,6) NOT NULL,
  `altitude` decimal(10,6) NOT NULL,
  `accuracy` float NOT NULL,
  `time` time NOT NULL,
  `date` date NOT NULL,
  `serial` varchar(45) NOT NULL,
  `hash` char(64) DEFAULT NULL,
  PRIMARY KEY (`timestamp`),
  UNIQUE KEY `uTime_UNIQUE` (`timestamp`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

###### 4. PHP Scripts
/var/www/mobilewitness

Folder structure for scripts is as follows:
```
+document root
|-config.ini            (database access ini file outside of www-data readable folder)
|-html
 |-uploadGPS.php        (write gps coordinates to database with optional hash e-mailing)
 |-uploadFile.php       (controls where uploaded files are saved to)
 |-uploads              (these folders store uploaded files)
  |-Audio
  |-Image
  |-Video
```

config.ini:
```
[database]
username = mobilewitness_user
password = mobilewitness_password
dbname = mobilewitness_database
```

uploadGPS.php:
```
<?php
        class GeolocateController {

                public function handleCoords($glob) {
                        if (isset($glob['latitude'], $glob['longitude'], $glob['altitude'], $glob['time'], $glob['serial'], $glob['accuracy'])) {

                                $config = parse_ini_file('../config.ini');
                                $username = $config['username'];
                                $password = $config['password'];
                                $dbname = $config['dbname'];
                                $host="localhost";

                                $latitude=$glob["latitude"];
                                $longitude=$glob["longitude"];
                                $altitude=$glob["altitude"];
                                $accuracy=$glob["accuracy"];
                                $time=$glob["time"];
                                $dDate=substr($time,0,10);
                                $dTime=substr($time,11,8);
                                $serial=$glob["serial"];

                                $con = new mysqli($host,$username,$password,$dbname);
                                if ($con->connect_errno) {
                                        echo "Failed to connect to MySQL: (" . $con->connect_errno . ") " . $con->connect_error;
                                }
                                if (!($stmt = $con->prepare("INSERT INTO `recordings` (latitude,longitude,altitude,accuracy,time,date,serial) VALUES (?,?,?,?,?,?,?)"))) {
                                        echo "Prepare failed: (" . $con->errno . ") " . $con->error;
                                }
                                if (!$stmt->bind_param("dddssss", $latitude,$longitude,$altitude,$accuracy,$dTime,$dDate,$serial)) {
                                        echo "Binding parameters failed: (" . $stmt->errno . ") " . $stmt->error;
                                }
                                if (!$stmt->execute()) {
                                        echo "Execute failed: (" . $stmt->errno . ") " . $stmt->error;
                                }
                                $stmt->close();
                                $con->close();
                        }
                }
}

        $geo = new GeolocateController;
        if (!empty($_POST)) {
        $response = $geo->handleCoords($_POST);
        }
        if ($response) {
                echo "success";
                } else {
                echo "failure";
        }
?>

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
                }

                move_uploaded_file($temp_name, $location.$file_name);
                echo "Uploaded";
        } else{
                echo "Error";
        }
?>
```
