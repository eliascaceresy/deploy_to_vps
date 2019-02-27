# Setup VPS

### 1. Add User Deploy
- root# apt-get update
- root# apt-get upgrade
- root# adduser deploy
- root# gpasswd -a deploy sudo
- root# sudo nano /etc/ssh/sshd_config
- root# PermitRootLogin no
- SAVE: ctrl + x, y, enter
- root# service ssh restart
- root# exit
- ssh deploy@server

### 2. Setup Languaje
- deploy# export LANGUAGE=en_US.UTF-8
- deploy# export LANG=en_US.UTF-8
- deploy# export LC_ALL=en_US.UTF-8
- deploy# export LC_CTYPE="en_US.UTF-8"
- deploy# locale-gen en_US.UTF-8
- deploy# sudo dpkg-reconfigure locales

### 3. Dependencies for ruby

- deploy# curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
- deploy# curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
- deploy# echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

- deploy# sudo apt-get update
- deploy# sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev nodejs yarn
#### 3.1 If you are using Redis
- deploy# sudo apt-get install redis-server
- deploy# sudo systemctl enable redis-server.service

### 4. Node.js
- deploy# curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
- deploy# sudo apt-get install -y nodejs

### 5. Install Ruby with RVM & Rails
- deploy# sudo apt-get install libgdbm-dev libncurses5-dev automake libtool bison libffi-dev
- deploy# gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
- deploy# curl -sSL https://get.rvm.io | bash -s stable
- deploy# source ~/.rvm/scripts/rvm
- deploy# rvm install 2.6.1
- deploy# rvm use 2.6.1 --default
- deploy# ruby -v
- deploy# gem install bundler
- deploy# gem install rails -v 5.2.2
- deploy# rails -v

### 6. Setup PostgreSQL 9.6
- deploy# sudo sh -c "echo 'deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main' > /etc/apt/sources.list.d/pgdg.list"
- deploy# wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -
- deploy# sudo apt-get update
- deploy# sudo apt-get install postgresql-common
- deploy# sudo apt-get install postgresql-9.6 libpq-dev
- deploy# sudo -u postgres createuser deploy -s
- deploy# sudo -u postgres psql
- postgres=# \password deploy
- postgres=# \q

### 7. Setup Nginx & Puma
- deploy# sudo add-apt-repository ppa:nginx/stable
- deploy# sudo apt-get update
- deploy# sudo apt-get -y install nginx
- deploy# sudo rm /etc/nginx/sites-available/default
- deploy# systemctl enable nginx.service
- deploy# wget https://raw.githubusercontent.com/puma/puma/master/tools/jungle/upstart/puma-manager.conf
- deploy# wget https://raw.githubusercontent.com/puma/puma/master/tools/jungle/upstart/puma.conf
- deploy# sudo nano puma.conf
- SET:
- setuid deploy
- setgid deploy
- SAVE: ctrl + x, y, enter
- deploy# sudo cp puma.conf puma-manager.conf /etc/init
- deploy# sudo touch /etc/puma.conf

### 8. Setup Certbot
- deploy# sudo apt-get install software-properties-common
- deploy# sudo add-apt-repository ppa:certbot/certbot
- deploy# sudo apt-get update
- deploy# sudo apt-get install certbot
