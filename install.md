# Installer le projet
## Installation des outils
### install aptitude
```shell
   sudo apt-get install aptitude
```
### install RVM with Ruby
```shell
  gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
```
```shell
  curl -sSL https://get.rvm.io | bash -s stable --ruby
```
### use ruby 2.2.0 as default version
For RVM to work properly, you have to check the 'Run command as login shell' checkbox on the Title and Command tab of gnome-terminal's
Edit ▸ Profile Preferences menu dialog.
```shell
  rvm install ruby_2.2.0
```
```shell
  rvm --default use 2.2.0
```
### install postgresql and postgresql dev
```shell
  sudo aptitude install postgresql
```
```shell
  sudo aptitude install postgresql-server-dev-9.5
```
### install bundler
```shell
  gem install bundler
```
```shell
  bundle install
```
### install nodejs
```shell
  sudo aptitude install nodejs
```
### install git
```shell
  sudo aptitude install git
```
### generating a new SSH key
When you're prompted to "Enter a file in which to save the key," press Enter. This accepts the default file location.
The same for passphrase, keep it empty then press Enter.
```shell
  ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
```shell
  ssh-add ~/.ssh/id_rsa
```
To add the ssh key to your git account, copies the contents of the id_rsa.pub file to your clipboardd
```shell
  cat ~/.ssh/id_rsa.pub
```
And then follow this tutorial from step 2 : https://help.github.com/articles/adding-a-new-ssh-key-to-your-github-account/

### clonning the project into your workspace
creat a new folder "dev" in your home
```shell
  cd ~
```
```shell
  mkdir dev

```
```shell
  cd dev

```
clone the project
```shell
  git clone git@github.com:Depannologue/mainsite.git

```
```shell
  mv mainsite depannologue
```
### install and setup nginx
```shell
  sudo aptitude install nginx
```
```shell
  sudo vim /etc/nginx/sites-enabled/default
```
replace the existing code by the following code and replace USER_NAME by your username
```shell

upstream unicorn {
  server unix:/tmp/unicorn.depannologue.sock fail_timeout=0;
}

server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name www.depannologue.dev_;
	return 301 https://$host$request_uri;
}

server {
  listen 443 ssl default;

  ssl on;
  ssl_certificate /home/YOUR_USERNAME/dev/depannologue/ssl/depannologue.crt;
  ssl_certificate_key /home/YOUR_USERNAME/dev/depannologue/ssl/depannologue.key;

  server_name www.depannologue.dev admin.depannologue.dev pro.depannologue.dev;
  root /home/YOUR_USERNAME/dev/depannologue/public;

  location ~ ^/(robots.txt|sitemap.xml.gz)/ {
    root /home/YOUR_USERNAME/dev/depannologue/public;
  }

  try_files $uri/index.html $uri @unicorn;
  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;
  keepalive_timeout 10;
}


```
### create database
```
  sudo -u postgres psql
```
make sure that your role name is the same as your username
```
  create role USER_NAME with login;
```
```
  create database depannologue_dev;  
```
```
  \q
```
```
  bundle exec rake db:schema:load
```
###  unicorn config
```
  vi ~/dev/depannologue/config/unicorn/development.rb
```
replace the existing code by the following code and change USER_NAME by your username
```
root = "/home/USER_NAME/dev/depannologue"
working_directory "#{root}"

pid "#{root}/tmp/pids/unicorn.pid"

stderr_path "#{root}/log/unicorn.log"
stdout_path "#{root}/log/unicorn.log"

worker_processes Integer(ENV['WEB_CONCURRENCY'] || 1)
timeout 30
preload_app true

listen '/tmp/unicorn.depannologue.sock', backlog: 64

before_fork do |server, worker|
Signal.trap 'TERM' do
  puts 'Unicorn master intercepting TERM and sending myself QUIT instead'
  Process.kill 'QUIT', Process.pid
end

defined?(ActiveRecord::Base) and
  ActiveRecord::Base.connection.disconnect!
end

after_fork do |server, worker|
Signal.trap 'TERM' do
  puts 'Unicorn worker intercepting TERM and doing nothing. Wait for master to send QUIT'
end

defined?(ActiveRecord::Base) and
  ActiveRecord::Base.establish_connection
end

# Force the bundler gemfile environment variable to
# reference the capistrano "current" symlink
before_exec do |_|
ENV['BUNDLE_GEMFILE'] = File.join(working_directory , 'Gemfile')
end

```
then
```
  sudo vi /etc/init.d/unicorn_depannologue
```
paste the code bellow and change USER_NAME by your username
```
#!/bin/sh

### BEGIN INIT INFO
# Provides:          unicorn
# Required-Start:    $all
# Required-Stop:     $all
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts the unicorn app server
# Description:       starts unicorn using start-stop-daemon
### END INIT INFO

set -e

USAGE="Usage: $0 <start|stop|restart|upgrade|rotate|force-stop>"

# app settings
USER="USER_NAME"
APP_NAME="depannologue"
APP_ROOT="/home/$USER/dev/$APP_NAME"
ENV="development"

# environment settings
CMD="cd $APP_ROOT  && bundle exec unicorn -c config/unicorn/development.rb -E $ENV -D"
PID="$APP_ROOT/tmp/pids/unicorn.pid"
OLD_PID="$PID.oldbin"

# make sure the app exists
cd $APP_ROOT || exit 1

sig () {
test -s "$PID" && kill -$1 `cat $PID`
}

oldsig () {
test -s $OLD_PID && kill -$1 `cat $OLD_PID`
}

case $1 in
start)
  sig 0 && echo >&2 "Already running" && exit 0
  echo "Starting $APP_NAME"
 su - $USER -c "$CMD"
  ;;
stop)
  echo "Stopping $APP_NAME"
  sig QUIT && exit 0
  echo >&2 "Not running"
  ;;
force-stop)
  echo "Force stopping $APP_NAME"
  sig TERM && exit 0
  echo >&2 "Not running"
  ;;
restart|reload|upgrade)
  sig USR2 && echo "reloaded $APP_NAME" && exit 0
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  $CMD
  ;;
rotate)
  sig USR1 && echo rotated logs OK && exit 0
  echo >&2 "Couldn't rotate logs" && exit 1
  ;;
*)
  echo >&2 $USAGE
  exit 1
  ;;
esac
```
```
update-rc.d unicorn_depannologue defaults
```
add the following code to your /etc/hosts file
```
sudo vi /etc/hosts
```
```
127.0.1.1       depanologue
127.0.0.1       www.depannologue.dev admin.depannologue.dev pro.depannologue.dev
```
