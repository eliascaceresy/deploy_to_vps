# Setup VPS

## 1. Add User Deploy

```bash
ssh root@ip_server

# Update packages
apt-get update

# Add deploy user
adduser deploy
gpasswd -a deploy sudo

# Disable Root Login
sudo nano /etc/ssh/sshd_config

# Find the followoing line and change by
PermitRootLogin no

# SAVE: ctrl + x, y, enter

# Restart ssh service
service ssh restart
exit

# Login with deploy user
ssh deploy@ip_server
```

## 2. Intall Ruby on Rails

```bash
# Add Node and Yarn APT Source List
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

# Update and Install necessary packages
sudo apt-get update
sudo apt-get install git-core zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl-openssl-dev software-properties-common libffi-dev nodejs yarn libgdbm-dev libncurses5-dev automake libtool bison libffi-dev

# Setup RVM
# GPG keys
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

# Install
curl -sSL https://get.rvm.io | bash -s stable
source ~/.rvm/scripts/rvm
rvm install 2.6.3
rvm use 2.6.3 --default

# Check current ruby version
ruby -v

# Install Rails
gem install bundler
gem install rails -v 'version'

# Check current rails version
rails -v
```

## 3. Setting Up PostgreSQL

```bash
# Add Postgresql Source List
echo 'deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main' > /etc/apt/sources.list.d/pgdg.list
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -

# Update packages and install
sudo apt-get update
sudo apt-get install -y postgresql-9.6 libpq-dev

# Create new postgres user
sudo -u postgres createuser 'user_name' -s
sudo -u postgres psql

# Set password
\password 'user_name'

#OBS. If you do not allow yourself to install postgresql, do it with `sudo su`
```

## 4. Add ssh to authorized keys in VPS

```bash
mkdir .ssh
nano ~/.ssh/authorized_keys

# Paste your ssh
# SAVE: ctrl + x, y, enter
```

## 5. Install Nginx

```bash
sudo apt-get update
sudo apt-get install curl nginx -y
```

## 6. Add ssh key to project git

```bash
ssh-keygen
cat ~/.ssh/id_rsa.pub

# In project git click Settings, click Deploy keys and add server ssh
ssh -T git@github.com
```

## 7. Capistrano Configuration

##### In `Gemfile.rb` add:

```ruby
group :development do
  gem 'capistrano',         require: false
  gem 'capistrano-rvm',     require: false
  gem 'capistrano-rails',   require: false
  gem 'capistrano-bundler', require: false
  gem 'capistrano3-puma',   require: false
  gem "capistrano-db-tasks", require: false
  gem 'capistrano-rails-db'
  gem 'capistrano-nginx'
  gem 'sshkit-sudo'
end
```

##### In `Capfile.rb` add:

```ruby
require "capistrano/setup"
require "capistrano/deploy"
require "capistrano/rails"
require "capistrano/bundler"
require "capistrano/rvm"
require 'capistrano/puma'
require "capistrano/scm/git"
require 'capistrano/rails/db'
require 'capistrano/nginx'
require 'sshkit/sudo'
require 'capistrano-db-tasks'

# If you are using seed_migration gem
require "capistrano/seed_migration_tasks"

# If you use Capistrano Local Precomplie (See Point 7.1)
# Add
require 'capistrano/local_precompile'
# Remove
require 'capistrano/rails/assets'

install_plugin Capistrano::SCM::Git
install_plugin Capistrano::Puma

Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```

#### 7.1. Capistrano Local Precompile [Gem](https://github.com/stve/capistrano-local-precompile "Capistrano Local Precompile")

One of the slowest parts of your deployment is waiting for asset pipeline precompilation. Capistrano Local Precompile takes a different approach. It builds your assets locally and rsync's them to your web server. So:

##### In `Gemfile.rb` add in `development's group`:

```ruby
  gem 'capistrano-local-precompile', '~> 1.2.0', require: false
```

##### setup

```ruby
bundle install
cap install
```

##### In `config/deploy.rb`

```ruby
server 'ip_server', roles: [:web, :app, :db], primary: true

set :application, 'app_name'
set :repo_url, 'repo_git'
set :user, 'deploy'
set :puma_threads, [4,16]
set :puma_workers, 0

set :pty,             true
set :use_sudo,        true

set :deploy_via,      :remote_cache
set :deploy_to,       "/home/#{fetch(:user)}/#{fetch(:application)}"
set :puma_bind,       "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock"
set :puma_state,      "#{shared_path}/tmp/pids/puma.state"
set :puma_pid,        "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma.error.log"
set :puma_error_log,  "#{release_path}/log/puma.access.log"
set :puma_preload_app, true
set :puma_worker_timeout, nil
set :puma_init_active_record, true  # Change to false when not using ActiveRecord
set :rvm_ruby_version, '2.6.3'

# If you have more than one environment, move this line to their respective config/deploy file
set :branch,        :branch_name

append :rvm_map_bins, 'rails'

# At this point in the deployment, these files must already be on the server, otherwise the deployment will fail
set :linked_files, %w{config/database.yml config/master.key}

set :linked_dirs,  %w{log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system public/uploads public/assets}

namespace :puma do
    desc 'Create Directories for Puma Pids and Socket'
    task :make_dirs do
        on roles(:app) do
            execute "mkdir #{shared_path}/tmp/sockets -p"
            execute "mkdir #{shared_path}/tmp/pids -p"
        end
    end
    before :start, :make_dirs
    after :make_dirs, 'nginx:restart'
    before :restart, 'nginx:restart'
end

namespace :deploy do

    desc 'Initial Deploy'
    task :initial do
        on roles(:app) do
            before 'deploy:restart', 'puma:start'
            invoke 'deploy'
        end
    end

    desc 'Restart application'
    task :restart do
        on roles(:app), in: :sequence, wait: 5 do
            invoke 'nginx:stop'
            invoke 'nginx:start'
            invoke 'puma:restart'
        end
    end

    before 'deploy:migrate', 'deploy:db:create'

    # If you are using seed_migration gem
    after 'deploy:migrate', 'seed:migrate'

    after  :finishing,    :compile_assets
    after  :finishing,    :cleanup
end
```

##### In `config/deploy/production.rb`

```ruby
set :stage, :production
set :rails_env, :production
server 'ip_server', user: 'deploy', roles: [:web, :app, :db], primary: true

# Capistrano Db Tasks
set :db_local_clean, true
set :db_remote_clean, true
set :disallow_pushing, false
set :db_ignore_data_tables, ["versions"]
```

##### Creating Nginx Server Block

In the server, inside /etc/nginx/site-avalidable/, create a file with name "app_name" or "domain_name" and paste the following

```conf
upstream app_name_puma_environment {
    server unix:///home/deploy/app_name/shared/tmp/sockets/app_name-puma.sock;
}

server {
    listen 80 default_server deferred;
    # si ya tienes un dominio colocarlo en la linea debajo, de lo contrario comentar
    server_name dominio.com;

    root /home/deploy/app_name/current/public;
    access_log /home/deploy/app_name/current/log/nginx.access.log;
    error_log /home/deploy/app_name/current/log/nginx.error.log info;

    try_files $uri/index.html $uri @app;

    location @app {
      proxy_pass        http://puma_app_name_environment;
      proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header  X-Real-IP       $remote_addr;
      proxy_set_header  X-Forwarded-Proto https;
      proxy_set_header  Host $http_host;
      proxy_redirect    off;
    }

    location /cable {
      proxy_pass http://puma_app_name_environment;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "Upgrade";
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-Proto https;
      proxy_redirect off;
    }

    location = /favicon.ico {
      log_not_found off;
      access_log    off;
    }

    location = /robots.txt {
      allow         all;
      log_not_found off;
      access_log    off;
    }

     location = /500.html {
      root /home/deploy/app_name/current/public;
    }

     location ~ /\.ht {
      deny  all;
    }

    location ^~ /assets/ {
      gzip_static on;
      expires max;
      add_header Cache-Control public;
    }

    access_log  /var/log/nginx/master.app_name.access.log;
    error_page 500 502 503 504 /500.html;
    client_max_body_size 4G;
    keepalive_timeout 10;
}

```

### 8. Extra protection

As an extra protection layer we will enable the ubuntu firewall, for this we must enter our server.

```bash
ssh deploy@ip_server
sudo ufw app list
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### 9. Push changes

Save and upload configurations given in step 7

### 10. Initial deploy

```bash
cap production deploy:initial
```

###### Note

- This first deploy should fail, since some files are missing on the server. For this we must create the following files:
- In home/deploy/app_name/shared/config `sudo nano database.yml` and paste de database production config
- In home/deploy/app_name/shared/config `sudo nano master.key` and paste the same key found in the local file config/master.key

### 11. Linked nginx

```bash
# Symbolic Link
sudo ln -nfs /etc/nginx/sites-available/app_or_domain_name /etc/nginx/sites-enabled/app_or_domain_name

# Restart service
sudo service nginx restart
```

### 12. Deploy

```bash
cap production deploy
```
