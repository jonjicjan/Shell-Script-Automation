# Shell-Script-Automation
# Introduction

This is a sample e-commerce application built for learning purposes.

Here's how to deploy it on CentOS systems:

## Deploy Pre-Requisites

1. Install FirewallD

```
sudo yum install -y firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo systemctl status firewalld
```

## Deploy and Configure Database

1. Install MariaDB

```
sudo yum install -y mariadb-server
sudo vi /etc/my.cnf
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

2. Configure firewall for Database

```
sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
sudo firewall-cmd --reload
```

3. Configure Database

```
$ mysql
MariaDB > CREATE DATABASE ecomdb;
MariaDB > CREATE USER 'ecomuser'@'localhost' IDENTIFIED BY 'ecompassword';
MariaDB > GRANT ALL PRIVILEGES ON *.* TO 'ecomuser'@'localhost';
MariaDB > FLUSH PRIVILEGES;
```

> ON a multi-node setup remember to provide the IP address of the web server here: `'ecomuser'@'web-server-ip'`

4. Load Product Inventory Information to database

Create the db-load-script.sql

```
cat > db-load-script.sql <<-EOF
USE ecomdb;
CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment,Name varchar(255) default NULL,Price varchar(255) default NULL, ImageUrl varchar(255) default NULL,PRIMARY KEY (id)) AUTO_INCREMENT=1;

INSERT INTO products (Name,Price,ImageUrl) VALUES ("Laptop","100","c-1.png"),("Drone","200","c-2.png"),("VR","300","c-3.png"),("Tablet","50","c-5.png"),("Watch","90","c-6.png"),("Phone Covers","20","c-7.png"),("Phone","80","c-8.png"),("Laptop","150","c-4.png");

EOF
```

Run sql script

```

sudo mysql < db-load-script.sql
```


## Deploy and Configure Web

1. Install required packages

```
sudo yum install -y httpd php php-mysqlnd
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --reload
```

2. Configure httpd

Change `DirectoryIndex index.html` to `DirectoryIndex index.php` to make the php page the default page

```
sudo sed -i 's/index.html/index.php/g' /etc/httpd/conf/httpd.conf
```

3. Start httpd

```
sudo systemctl start httpd
sudo systemctl enable httpd
```

4. Download code

```
sudo yum install -y git
sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/
```

<!-- 5. Update index.php

Update [index.php](https://github.com/kodekloudhub/learning-app-ecommerce/blob/13b6e9ddc867eff30368c7e4f013164a85e2dccb/index.php#L107) file to connect to the right database server. In this case `localhost` since the database is on the same server.

```
sudo sed -i 's/172.20.1.101/localhost/g' /var/www/html/index.php

              <?php
                        $link = mysqli_connect('172.20.1.101', 'ecomuser', 'ecompassword', 'ecomdb');
                        if ($link) {
                        $res = mysqli_query($link, "select * from products;");
                        while ($row = mysqli_fetch_assoc($res)) { ?>
```

> ON a multi-node setup remember to provide the IP address of the database server here.
```
sudo sed -i 's/172.20.1.101/localhost/g' /var/www/html/index.php
```
-->

5. Create and Configure the `.env` File

   Create an `.env` file in the root of your project folder.

   ```sh
   cat > /var/www/html/.env <<-EOF
   DB_HOST=localhost
   DB_USER=ecomuser
   DB_PASSWORD=ecompassword
   DB_NAME=ecomdb
   EOF

6. Update `index.php`

   Update the `index.php` file to load the environment variables from the `.env` file and use them to connect to the database.

   ```php
   <?php
   // Function to load environment variables from a .env file
   function loadEnv($path)
   {
       if (!file_exists($path)) {
           return false;
       }

       $lines = file($path, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
       foreach ($lines as $line) {
           if (strpos(trim($line), '#') === 0) {
               continue;
           }

           list($name, $value) = explode('=', $line, 2);
           $name = trim($name);
           $value = trim($value);
           putenv(sprintf('%s=%s', $name, $value));
       }
       return true;
   }

   // Load environment variables from .env file
   loadEnv(__DIR__ . '/.env');

   // Retrieve the database connection details from environment variables
   $dbHost = getenv('DB_HOST');
   $dbUser = getenv('DB_USER');
   $dbPassword = getenv('DB_PASSWORD');
   $dbName = getenv('DB_NAME');

   ?>

   ON a multi-node setup, remember to provide the IP address of the database server in the .env file.


7. Test

```
curl http://localhost
```
####script
#!/bin/bash
#
# Automate ECommerce Application Deployment
# Author: Mumshad Mannambeth

#######################################
# Print a message in a given color.
# Arguments:
#   Color. eg: green, red
#######################################
function print_color(){
  NC='\033[0m' # No Color

  case $1 in
    "green") COLOR='\033[0;32m' ;;
    "red") COLOR='\033[0;31m' ;;
    "*") COLOR='\033[0m' ;;
  esac

  echo -e "${COLOR} $2 ${NC}"
}

#######################################
# Check the status of a given service. If not active exit script
# Arguments:
#   Service Name. eg: firewalld, mariadb
#######################################
function check_service_status(){
  service_is_active=$(sudo systemctl is-active $1)

  if [ $service_is_active = "active" ]
  then
    echo "$1 is active and running"
  else
    echo "$1 is not active/running"
    exit 1
  fi
}

#######################################
# Check the status of a firewalld rule. If not configured exit.
# Arguments:
#   Port Number. eg: 3306, 80
#######################################
function is_firewalld_rule_configured(){

  firewalld_ports=$(sudo firewall-cmd --list-all --zone=public | grep ports)

  if [[ $firewalld_ports == *$1* ]]
  then
    echo "FirewallD has port $1 configured"
  else
    echo "FirewallD port $1 is not configured"
    exit 1
  fi
}

#######################################
# Check if a given item is present in an output
# Arguments:
#   1 - Output
#   2 - Item
#######################################
function check_item(){
  if [[ $1 = *$2* ]]
  then
    print_color "green" "Item $2 is present on the web page"
  else
    print_color "red" "Item $2 is not present on the web page"
  fi
}






echo "---------------- Setup Database Server ------------------"

# Install and configure firewalld
print_color "green" "Installing FirewallD.. "
sudo yum install -y firewalld

print_color "green" "Installing FirewallD.. "
sudo systemctl start firewalld
sudo systemctl enable firewalld

# Check FirewallD Service is running
check_service_status firewalld

# Install and configure Maria-DB
print_color "green" "Installing MariaDB Server.."
sudo yum install -y mariadb-server

print_color "green" "Starting MariaDB Server.."
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Check FirewallD Service is running
check_service_status mariadb

# Configure Firewall rules for Database
print_color "green" "Configuring FirewallD rules for database.."
sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
sudo firewall-cmd --reload

is_firewalld_rule_configured 3306


# Configuring Database
print_color "green" "Setting up database.."
cat > setup-db.sql <<-EOF
  CREATE DATABASE ecomdb;
  CREATE USER 'ecomuser'@'localhost' IDENTIFIED BY 'ecompassword';
  GRANT ALL PRIVILEGES ON *.* TO 'ecomuser'@'localhost';
  FLUSH PRIVILEGES;
EOF

sudo mysql < setup-db.sql

# Loading inventory into Database
print_color "green" "Loading inventory data into database"
cat > db-load-script.sql <<-EOF
USE ecomdb;
CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment,Name varchar(255) default NULL,Price varchar(255) default NULL, ImageUrl varchar(255) default NULL,PRIMARY KEY (id)) AUTO_INCREMENT=1;

INSERT INTO products (Name,Price,ImageUrl) VALUES ("Laptop","100","c-1.png"),("Drone","200","c-2.png"),("VR","300","c-3.png"),("Tablet","50","c-5.png"),("Watch","90","c-6.png"),("Phone Covers","20","c-7.png"),("Phone","80","c-8.png"),("Laptop","150","c-4.png");

EOF

sudo mysql < db-load-script.sql

mysql_db_results=$(sudo mysql -e "use ecomdb; select * from products;")

if [[ $mysql_db_results == *Laptop* ]]
then
  print_color "green" "Inventory data loaded into MySQl"
else
  print_color "green" "Inventory data not loaded into MySQl"
  exit 1
fi


print_color "green" "---------------- Setup Database Server - Finished ------------------"

print_color "green" "---------------- Setup Web Server ------------------"

# Install web server packages
print_color "green" "Installing Web Server Packages .."
sudo yum install -y httpd php php-mysqlnd

# Configure firewalld rules
print_color "green" "Configuring FirewallD rules.."
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --reload

is_firewalld_rule_configured 80

# Update index.php
sudo sed -i 's/index.html/index.php/g' /etc/httpd/conf/httpd.conf

# Start httpd service
print_color "green" "Start httpd service.."
sudo systemctl  start httpd
sudo systemctl enable httpd

# Check FirewallD Service is running
check_service_status httpd

# Download code
print_color "green" "Install GIT.."
sudo yum install -y git
sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/
sudo sed -i 's#// \(.*mysqli_connect.*\)#\1#' /var/www/html/index.php
sudo sed -i 's#// \(\$link = mysqli_connect(.*172\.20\.1\.101.*\)#\1#; s#^\(\s*\)\(\$link = mysqli_connect(\$dbHost, \$dbUser, \$dbPassword, \$dbName);\)#\1// \2#' /var/www/html/index.php

print_color "green" "Updating index.php.."
sudo sed -i 's/172.20.1.101/localhost/g' /var/www/html/index.php

print_color "green" "---------------- Setup Web Server - Finished ------------------"

# Test Script
web_page=$(curl http://localhost)

for item in Laptop Drone VR Watch Phone
do
  check_item "$web_page" $item
done


