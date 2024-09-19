```
#!/bin/bash

# Function to install on Debian/Ubuntu
install_debian_ubuntu() {
  echo "Installing on Debian/Ubuntu..."

  # Update and upgrade packages
  sudo apt update
  sudo apt upgrade -y

  # Install Nginx, PHP, MariaDB, and phpMyAdmin
  sudo apt install -y nginx php-fpm php-mysql mariadb-server phpmyadmin

  # Start and enable services
  sudo systemctl start nginx
  sudo systemctl enable nginx

  sudo systemctl start mariadb
  sudo systemctl enable mariadb

  # Configure phpMyAdmin with access control
  sudo bash -c 'cat > /etc/nginx/conf.d/phpmyadmin.conf <<EOF
server {
    listen 80;
    server_name _;

    root /usr/share/phpmyadmin;
    index index.php index.html index.htm;

    location / {
        try_files \$uri \$uri/ =404;
    }

    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        include fastcgi_params;
    }

    location /phpmyadmin {
        root /usr/share/;
        index index.php;
        allow all;
    }

    location ~ /\.ht {
        deny all;
    }
}
EOF'

  # Create symbolic link for phpMyAdmin
  sudo ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin

  # Secure MariaDB
  sudo mysql_secure_installation

  # Set permissions for phpMyAdmin
  sudo chown -R www-data:www-data /usr/share/phpmyadmin
  sudo chmod -R 755 /usr/share/phpmyadmin

  # Restart Nginx to apply changes
  sudo systemctl restart nginx

  echo "Installation and configuration completed on Debian/Ubuntu."
}

# Function to install on CentOS/Fedora
install_centos_fedora() {
  echo "Installing on CentOS/Fedora..."

  # Update packages
  sudo dnf update -y

  # Install Nginx, PHP, MariaDB, and phpMyAdmin
  sudo dnf install -y epel-release
  sudo dnf install -y nginx php php-mysqlnd mariadb-server phpMyAdmin

  # Start and enable services
  sudo systemctl start nginx
  sudo systemctl enable nginx

  sudo systemctl start mariadb
  sudo systemctl enable mariadb

  # Configure phpMyAdmin with access control
  sudo bash -c 'cat > /etc/nginx/conf.d/phpmyadmin.conf <<EOF
server {
    listen 80;
    server_name _;

    root /usr/share/phpMyAdmin;
    index index.php index.html index.htm;

    location / {
        try_files \$uri \$uri/ =404;
    }

    location ~ \.php\$ {
        include fastcgi_params;
        fastcgi_pass unix:/var/run/php-fpm/www.sock;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
    }

    location /phpmyadmin {
        root /usr/share/;
        index index.php;
        allow all;
    }

    location ~ /\.ht {
        deny all;
    }
}
EOF'

  # Create symbolic link for phpMyAdmin
  sudo ln -s /usr/share/phpMyAdmin /var/www/html/phpmyadmin

  # Secure MariaDB
  sudo mysql_secure_installation

  # Set permissions for phpMyAdmin
  sudo chown -R nginx:nginx /usr/share/phpMyAdmin
  sudo chmod -R 755 /usr/share/phpMyAdmin

  # Handle SELinux
  if [ "$(getenforce)" = "Enforcing" ]; then
    sudo setsebool -P httpd_can_network_connect_db 1
    sudo setsebool -P httpd_can_network_connect 1
    sudo restorecon -Rv /usr/share/phpMyAdmin
  fi

  # Restart Nginx to apply changes
  sudo systemctl restart nginx

  echo "Installation and configuration completed on CentOS/Fedora."
}

# Function to create MariaDB user
create_mariadb_user() {
  read -p "Enter the MariaDB root password: " root_password
  read -p "Enter the new MariaDB username: " username
  read -s -p "Enter the password for the new user: " user_password
  echo

  # Create the user in MariaDB
  sudo mysql -u root -p"$root_password" -e "CREATE USER '$username'@'localhost' IDENTIFIED BY '$user_password';"
  sudo mysql -u root -p"$root_password" -e "GRANT ALL PRIVILEGES ON *.* TO '$username'@'localhost' WITH GRANT OPTION;"
  sudo mysql -u root -p"$root_password" -e "FLUSH PRIVILEGES;"

  echo "MariaDB user '$username' created successfully."
}

# Check the Linux distribution
if [ -f /etc/debian_version ]; then
  install_debian_ubuntu
elif [ -f /etc/redhat-release ]; then
  install_centos_fedora
else
  echo "Unsupported Linux distribution."
  exit 1
fi

# Create a MariaDB user
create_mariadb_user
```
