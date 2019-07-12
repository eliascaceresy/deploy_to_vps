# Setup VPS

### 1. Add User Deploy

- `ssh root@ip_server`
- `root# apt-get update`
- `root# adduser deploy`
- `root# gpasswd -a deploy sudo`
- `root# sudo nano /etc/ssh/sshd_config`
- `root# PermitRootLogin no`
- SAVE: ctrl + x, y, enter
- `root# service ssh restart`
- `root# exit`
- `ssh deploy@ip_server`

### 2. Intall Ruby on Rails

- `curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -`
- `curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -`
- `echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list`

- `sudo apt-get update`
- `sudo apt-get install git-core zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev nodejs yarn`
- `sudo apt-get install libgdbm-dev libncurses5-dev automake libtool bison libffi-dev`
- `gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB`
- `curl -sSL https://get.rvm.io | bash -s stable`
- `source ~/.rvm/scripts/rvm`
- `rvm install 2.6.3`
- `rvm use 2.6.3 --default`
- `ruby -v`
- `gem install bundler`
- `gem install rails -v 6.0.0.rc1`
- `rails -v`

### 3. Setting Up PostgreSQL

- `echo 'deb http://apt.postgresql.org/pub/repos/apt/ xenial-pgdg main' > /etc/apt/sources.list.d/pgdg.list`
- `wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -`
- `sudo apt-get update`
- `sudo apt-get install -y postgresql-9.6 libpq-dev`
- `sudo -u postgres createuser user_name -s`
- `sudo -u postgres psql`
- postgres=# `\password user_name`
- obs. If you do not allow yourself to install postgresql, do it with `sudo su`

### 4. Add ssh to authorized keys in VPS

- `mkdir .ssh`
- `nano ~/.ssh/authorized_keys`
- Paste your ssh
- SAVE: ctrl + x, y, enter

### 5. Install Nginx

- `sudo apt-get update`
- `sudo apt-get install curl git-core nginx -y`

### 6. Add ssh key to project git

- `ssh-keygen`
- `cat ~/.ssh/id_rsa.pub`
- In project git click Settings, click Deploy keys and add server ssh
- `ssh -T git@github.com`

### 7. Capistrano Configuration

- In gemfile add:

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

- `bundle install`
- `cap install`

- in Capfile

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

install_plugin Capistrano::SCM::Git
install_plugin Capistrano::Puma

Dir.glob("lib/capistrano/tasks/*.rake").each { |r| import r }
```

- In config/deploy.rb

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
set :deploy_to,       "/home/#{fetch(:user)}/apps/#{fetch(:application)}"
set :puma_bind,       "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock"
set :puma_state,      "#{shared_path}/tmp/pids/puma.state"
set :puma_pid,        "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma.error.log"
set :puma_error_log,  "#{release_path}/log/puma.access.log"
set :puma_preload_app, true
set :puma_worker_timeout, nil
set :puma_init_active_record, true  # Change to false when not using ActiveRecord
set :rvm_ruby_version, '2.6.3'

set :branch,        :branch_name

append :rvm_map_bins, 'rails'

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

- In config/deploy/production.rb

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

- Inside config folder, create a file nginx.conf and paste the following

```conf
upstream puma {
    server unix:///home/deploy/apps/app_name/shared/tmp/sockets/app_name-puma.sock;
}

server {
    listen 80 default_server deferred;
    # si ya tienes un dominio colocarlo en la linea debajo, de lo contrario comentar
    server_name dominio.com;

    root /home/deploy/apps/app_name/current/public;
    access_log /home/deploy/apps/app_name/current/log/nginx.access.log;
    error_log /home/deploy/apps/app_name/current/log/nginx.error.log info;

    location ^~ /assets/ {
        gzip_static on;
        expires max;
        add_header Cache-Control public;
    }

    try_files $uri/index.html $uri @puma;
    location @puma {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect off;

        proxy_pass http://puma;
    }

    error_page 500 502 503 504 /500.html;
    client_max_body_size 10M;
    keepalive_timeout 10;
}

```

### 8. Extra protection

- As an extra protection layer we will enable the ubuntu firewall, for this we must enter our server.
- `ssh deploy@ip_server`
- `sudo ufw app list`
- `sudo ufw allow OpenSSH`
- `sudo ufw allow 'Nginx Full'`
- `sudo ufw enable`

### 9. Puch changes

- Save and upload configurations given in step 7

### 10. Initial deploy

- `cap production deploy:initial`
- This first deploy should fail, since some files are missing on the server. For this we must create the following files:
- In home/deploy/apps/app_name/shared/config `sudo nano database.yml` and paste de database production config
- In home/deploy/apps/app_name/shared/config `sudo nano master.key` and paste the same key found in the local file config/master.key

### 11. Linked nginx

- `sudo rm /etc/nginx/sites-enabled/default`
- `sudo ln -nfs /home/deploy/apps/app_name/current/config/nginx.conf /etc/nginx/sites-enabled/app_name`
- `sudo service nginx restart`

### 12. Deploy

- `cap production deploy`
