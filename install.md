# Install the project

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
  rvm install ruby-2.2.0
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
```shell
  cd ~/dev/depannologue
```
```shell
  bundle install
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
```
  create user admin with superuser;
```
```
  \q
```
set admin as a default username of the database
```
  vi ~/dev/depannologue/config/database.yml
```
change the existing code to look like follow
```
development:
host: localhost
adapter: postgresql
encoding: unicode
database: depannologue_dev
username: admin
pool: 5

production:
adapter: postgresql
encoding: unicode
database: depannologue_production
username: <%= Rails.application.secrets[:database_username] %>
password: <%= Rails.application.secrets[:database_password] %>
pool: 5
```
you have to change  the postgres config file to use rails command,
```
sudo vi /etc/postgresql/9.5/main/pg_hba.conf

```
change the existing code to look like the following code

```
local   all             postgres                              md5

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             localhost               trust
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                peer
#host    replication     postgres        127.0.0.1/32            md5
#host    replication     postgres        ::1/128                 md5

```
now we can create the database using rails commands
```
  bundle exec rake db:create
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
create tmp folder
```
cd ~/dev/depannologue
```
```
mkdir -p tmp/{cache,data/meta_request,pids,sessions,sockets}
```
create secret.yml
```
sudo vi ~/dev/depannologue/config/secret.yml
```
paste the following code
```
# Be sure to restart your server when you modify this file.

# Your secret key is used for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!

# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
# You can use `rake secret` to generate a secure secret key.

# Make sure the secrets in this file are kept private
# if you're sharing your code publicly.

development:
  secret_key_base:    3be4eb24787cc652b8991aad318291e4474cf707a22fb47ad92dfa05e65722a1aef7212abcc60f11fd2679017689c549622b90111729d36ce7dde1e108856678
  twilio_account_sid: AC57e156e38e25920c20599ae15549ff0f
  twilio_auth_token:  f40f003064270f145fdffdf91bce2102
  twilio_from:        33644605079
  default_from:       noreply@depannologue.com
  host:               http://localhost:3000
  lockup_actived:     false
  lockup_codeword:    love
  mailgun_api_key:    key-90f2790da18c8f7a93bbf7999ec6c77f
  mailgun_domain:     sandbox2ceff1ee93d84edf9664f89fc67944b4.mailgun.org
```
### install sidekiq
```
 cd ~/depannologue
```
```
 gem install sidekiq
```
```
  sudo vi /etc/init.d/
```
paste the following code and change USER_NAME by your USER_NAME
```
#!/bin/bash
# sidekiq    Init script for Sidekiq
# chkconfig: 345 100 75
#
# Description: Starts and Stops Sidekiq message processor for Stratus application.
#
# User-specified exit parameters used in this script:
#
# Exit Code 5 - Incorrect User ID
# Exit Code 6 - Directory not found


# You will need to modify these
APP="depannologue"
AS_USER="USER_NAME"
APP_DIR="/home/USER_NAME/dev/${APP}"

APP_CONFIG="${APP_DIR}/config"
LOG_FILE="$APP_DIR/log/sidekiq.log"
LOCK_FILE="$APP_DIR/${APP}-lock"
PID_FILE="$APP_DIR/${APP}.pid"
GEMFILE="$APP_DIR/Gemfile"
SIDEKIQ="sidekiq"
APP_ENV="development"
BUNDLE="bundle"

START_CMD="$BUNDLE exec $SIDEKIQ -e $APP_ENV -P $PID_FILE"
CMD="cd ${APP_DIR}; ${START_CMD} >> ${LOG_FILE} 2>&1 &"

RETVAL=0


start() {

  status
  if [ $? -eq 1 ]; then

    [ `id -u` == '0' ] || (echo "$SIDEKIQ runs as root only .."; exit 5)
    [ -d $APP_DIR ] || (echo "$APP_DIR not found!.. Exiting"; exit 6)
    cd $APP_DIR
    echo "Starting $SIDEKIQ message processor .. "

    su -c "$CMD" - $AS_USER

    RETVAL=$?
    #Sleeping for 8 seconds for process to be precisely visible in process table - See status ()
    sleep 8
    [ $RETVAL -eq 0 ] && touch $LOCK_FILE
    return $RETVAL
  else
    echo "$SIDEKIQ message processor is already running .. "
  fi


}

stop() {

    echo "Stopping $SIDEKIQ message processor .."
    SIG="INT"
    kill -$SIG `cat  $PID_FILE`
    RETVAL=$?
    [ $RETVAL -eq 0 ] && rm -f $LOCK_FILE
    return $RETVAL
}

status() {

  ps -ef | grep 'sidekiq [0-9].[0-9].[0-9]' | grep -v grep
  return $?
}


case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status

        if [ $? -eq 0 ]; then
             echo "$SIDEKIQ message processor is running .."
             RETVAL=0
         else
             echo "$SIDEKIQ message processor is stopped .."
             RETVAL=1
         fi
        ;;
    *)
        echo "Usage: $0 {start|stop|status}"
        exit 0
        ;;
esac
exit $RETVAL
```

```
  sudo update-rc init_sidekiq.sh defaults
```
### install redis
```
sudo aptitude install redis-server
```
### clockwork config
```
  sudo vi /etc/init.d init_clockwork
```
```
  sudo vi /etc/init.d init_clockwork
```
```
#!/bin/sh

### BEGIN INIT INFO
# Provides:          clockwork
# Required-Start:    $all
# Required-Stop:     $all
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts the clockwork service
# Description:       starts clockwork service
### END INIT INFO
APP="depannologue"
AS_USER="depanologue"
APP_DIR="/home/depanologue/dev/${APP}"

APP_CONFIG="${APP_DIR}/config"
LOG_FILE="$APP_DIR/log/clockwork.log"
LOCK_FILE="$APP_DIR/${APP}-lock"
PID_FILE="$APP_DIR/${APP}.pid"
UNICORN_PID="$APP_DIR/tmp/pids/unicorn.pid"
GEMFILE="$APP_DIR/Gemfile"
CLOCKWORK="clockwork"
APP_ENV="development"
BUNDLE="bundle"

START_CMD="$BUNDLE exec $CLOCKWORK ${APP_DIR}/lib/clock.rb "
CMD="cd ${APP_DIR}; ${START_CMD} >> ${LOG_FILE} 2>&1 &"

RETVAL=0


start() {

  status
  if [ $? -eq 1 ]; then

    [ `id -u` == '0' ] || (echo "$CLOCKWORK runs as root only .."; exit 5)
    [ -d $APP_DIR ] || (echo "$APP_DIR not found!.. Exiting"; exit 6)
    cd $APP_DIR
    echo "Starting $CLOCKWORK message processor .. "

    su -c "$CMD" - $AS_USER

    RETVAL=$?
    #Sleeping for 8 seconds for process to be precisely visible in process table - See status ()
    sleep 8
    [ $RETVAL -eq 0 ]
    return $RETVAL
  else
    echo "$CLOCKWORK message processor is already running .. "
  fi


}

stop() {
    TIMEOUT=120
    echo "Stopping $CLOCKWORK message processor .."
    n=$TIMEOUT
    echo 'Waiting for unicorn master to stop...'
    while [ -s "$UNICORN_PID" ] && [ "$n" -ge 0 ]
    do
      sleep 1 && n=$(( $n - 1 ))
    done
    SIG="INT"
    kill -$SIG `cat  $PID_FILE`
    RETVAL=$?
    [ $RETVAL -eq 0 ] && rm -f $LOCK_FILE
    return $RETVAL
}

status() {

  ps -ef | grep 'clockwork [0-9].[0-9].[0-9]' | grep -v grep
  return $?
}


case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status

        if [ $? -eq 0 ]; then
             echo "$CLOCKWORK message processor is running .."
             RETVAL=0
         else
             echo "$CLOCKWORK message processor is stopped .."
             RETVAL=1
         fi
        ;;
    *)
        echo "Usage: $0 {start|stop|status}"
        exit 0
        ;;
esac
exit $RETVAL
```
```
  sudo update-rc init_clockwork defaults
```
