# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "codementality/centos7-docker"
  config.ssh.insert_key = false
  # config.vbguest.auto_update = false
  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # NOTE: This will enable public access to the opened port
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.160"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder ".", "/var/www/html", type: "nfs"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
    vb.memory = "8192"
  end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.

  config.vm.post_up_message = "This is the start up message!"

  config.vm.provision "shell", inline: <<-SHELL
    curl 'https://setup.ius.io/' -o setup-ius.sh
    bash setup-ius.sh
    su -c 'yum -y update'

    ## Install MariaDB and secure the installation
    yum install -y mariadb mariadb-server -y
    systemctl enable mariadb.service
    systemctl start mariadb
    mysql -u root -e "SET PASSWORD FOR 'root'@'localhost' = PASSWORD('pass'); \
      DROP DATABASE test; \
      SET PASSWORD FOR 'root'@'127.0.0.1' = PASSWORD('pass'); \
      SET PASSWORD FOR 'root'@'::1' = PASSWORD('pass'); \
      SET PASSWORD FOR 'root'@'localhost.localdomain' = PASSWORD('pass'); \
      DELETE FROM mysql.user where User=''; \
      DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1'); \
      FLUSH PRIVILEGES;"

    ## Install memcached
    yum install -y memcached
    systemctl enable memcached.service
    systemctl start memcached

    ## Install Apache
    yum install -y httpd
    ## Change Apache user and group
    sed -i -e 's/User apache/User vagrant/g' /etc/httpd/conf/httpd.conf
    sed -i -e 's/Group apache/Group vagrant/g' /etc/httpd/conf/httpd.conf
    systemctl enable httpd.service

    ## Install PHP
    yum install -y mod_php70u php70u-cli php70u-mysqlnd php70u-bcmath \
      php70u-common php70u-gd php70u-intl php70u-ldap php70u-json \
      php70u-mbstring php70u-mcrypt php70u-pdo php70u-pecl-apcu \
      php70u-pecl-memcached php70u-xdebug php70u-soap php70u-xml
    ## Increase php memory limit to 256M (local only, prod should be 192M)
    sed -i -e 's/memory_limit = 128M/memory_limit = 256M/g' /etc/php.ini
    systemctl restart httpd.service

    ## Install Composer
    # Composer version
    export COMPOSER_HOME=/composer
    export PATH=/composer/vendor/bin:$PATH
    export COMPOSER_ALLOW_SUPERUSER=1
    export COMPOSER_VERSION=1.3.0

    # Setup the Composer installer
    curl -o /tmp/composer-setup.php https://getcomposer.org/installer
    curl -o /tmp/composer-setup.sig https://composer.github.io/installer.sig
    php -r "if (hash('SHA384', file_get_contents('/tmp/composer-setup.php')) !== trim(file_get_contents('/tmp/composer-setup.sig'))) { unlink('/tmp/composer-setup.php'); echo 'Invalid installer' . PHP_EOL; exit(1); }"

    # Install Composer
    php /tmp/composer-setup.php --no-ansi --install-dir=/usr/local/bin --filename=composer --version=${COMPOSER_VERSION} && rm -rf /tmp/composer-setup.php
    # Install Prestissimo plugin for Composer -- allows for parallel processing of packages during install / update
    composer global require "hirak/prestissimo:^0.3" --optimize-autoloader
    chown -Rf vagrant:vagrant /composer

    # Install Drush
    mkdir /usr/local/drush
    cd /usr/local/drush
    composer init --require=drush/drush:8.* -n
    composer config bin-dir /usr/local/bin
    composer install
    drush init -y
    # Set up default location for drush alias files
    mkdir -p /etc/drush/site-aliases
    echo  "<?php \
$aliases[isset($_SERVER['dev'] = [ \
  'root' => '/var/www/html', \
  'uri' => 'localhost', \
];" >> /etc/drush/site-aliases/default.aliases.drushrc.php

    ## Install Drupal Console
    composer require drupal/console-en:^1.0.0-rc26 drupal/console-core:^1.0.0-rc26 \
      drupal/console:^1.0.0-rc26 --optimize-autoloader

    ## Setting up local ownership
    chown -Rf vagrant:vagrant /var/www
    ## Set up virtual_host capabilities, directory and modify httpd.conf
    mkdir -p /etc/httpd/sites-available
    echo "IncludeOptional sites-available/*.conf" >> /etc/httpd/conf/httpd.conf
    ## Create VM file
    echo "<VirtualHost *:80>
  ServerName localhost
  <Directory  "/var/www/html">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
    RewriteEngine on
    RewriteBase /
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_URI} !=/favicon.ico
    RewriteRule ^ index.php [L]
  </Directory>
  DocumentRoot /var/www/html
</VirtualHost>" > /etc/httpd/sites-available/default.conf

    ## Restart Apache
    systemctl restart httpd.service
    yum clean all
    dd if=/dev/zero of=/EMPTY bs=1M
    sudo rm -f /EMPTY
    cat /dev/null > ~/.bash_history && history -c && exit
  SHELL

end
