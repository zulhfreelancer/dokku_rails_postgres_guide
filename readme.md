### How to deploy Rails + Postgresql app to Ubuntu VPS using Dokku

In this example, I'm using Ubuntu 16.04 LTS, Ruby 2.3.1, Rails 4.2.6, Dokku 0.6.4 and [dokku-postgres plugin](https://github.com/dokku/dokku-postgres).

#### STEP 0: Important

Please change this values to your own values:

1. `IP_ADDRESS`
2. `APP_NAME`
3. `APP_DB_NAME`
4. `THE_DOMAIN_NAME`

----

#### STEP 1: Server Setup
1. spin up a new VPS server
2. local: `ssh root@IP_ADDRESS`
3. vps: change the root password

#### STEP 2: Add SSH Key to Server
1. local: `ssh-copy-id root@IP_ADDRESS`
2. login again and make sure no password prompt showing up
3. vps: `nano /etc/ssh/sshd_config`
4. locate `PermitRootLogin` and update it to `PermitRootLogin without-password`
5. vps: `ps auxw | grep ssh`
6. locate process that has `/usr/sbin/sshd -D` inside the COMMAND column (last column) and take the PID (second column from left)
7. vps: `kill -HUP THE_PID_NUMBER`

#### STEP 3: Install Dokku
1. vps: `wget https://raw.githubusercontent.com/dokku/dokku/v0.6.4/bootstrap.sh` - please change the version number (0.6.4) to latest version
2. vps: `nano /etc/ssh/ssh_config`
3. Add this:
```
ServerAliveInterval 300
```
4. vps: `sudo DOKKU_TAG=v0.6.4 bash bootstrap.sh`
5. If get _E: Unable to lock the ... directory (/var/lib/dpkg/) is another process using it?_ error, run this on vps: `ps aux | grep apt`
6. take all PID numbers (second column from left)
7. vps: `kill THE_PID_NUMBER` (do it one by one)
8. If get _E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'_ error, just run this: `sudo dpkg --configure -a`
9. Re-run this on vps: `sudo DOKKU_TAG=v0.6.4 bash bootstrap.sh`

#### STEP 4: Install Ruby (RVM) & Bundler Gem
1. vps: `curl -L https://get.rvm.io | bash -s stable`
2. If get _GPG signature verification failed_ error, run this on vps: `gpg --keyserver hkp://keys.gnupg.net --recv-keys THE_STRING_YOU_SEE_ON_THE_SCREEN`
3. Re-run on vps: `curl -L https://get.rvm.io | bash -s stable`
4. vps: `source /etc/profile.d/rvm.sh`
5. vps: `rvm install 2.3.1`
6. vps: `gem install bundler`

#### STEP 5: Finishing Dokku Installation
1. open VPS IP_ADDRESS in browser
2. local: `cat ~/.ssh/id_rsa.pub`
3. paste inside the textarea box
4. put VPS IP_ADDRESS inside the Hostname field
5. leave the _Use virtualhost naming for apps_ checkbox uncheck
6. click _Finish Setup_ button

#### STEP 6: Install Postgres Plugin & Create a Database
1. vps: `dokku plugin:install https://github.com/dokku/dokku-postgres.git postgres`
2. vps: `dokku postgres:create APP_DB_NAME`

#### STEP 7: Prepare to Deploy
1. add `gem 'pg'` inside Gemfile of the project
2. create a Procfile at project root and add this line:
`web: bundle exec rails server -p $PORT -e $RACK_ENV`
3. make sure you already change database adapter inside **database.yml** to `postgresql` and `gem 'sqlite3'` is not in the production group gems
4. bundle install, git add and commit everything
5. local: `git remote add dokku dokku@IP_ADDRESS:APP_NAME`

#### STEP 8: Deploy
1. local: `git push dokku master`
2. if get _remote rejected_ error during push, run this on vps:
`dokku config:set APP_NAME CURL_TIMEOUT=600` and `git push dokku master` again
3. also can try run this on vps: `dokku config:set APP_NAME BUILDPACK_URL=https://github.com/heroku/heroku-buildpack-ruby.git`
4. if get _DOKKU SCALE: No such file or directory_ error during push, create a file name `DOKKU_SCALE` in /home/dokku/APP_NAME and add this:
```
web=1
worker=0
```

#### STEP 9: Link Database with App and DB Migrate
1. vps: `cd /home/dokku/APP_NAME`
2. vps: `dokku postgres:link APP_DB_NAME APP_NAME`
3. vps: `dokku run APP_NAME rake db:migrate`

#### STEP 10: Using Custom Domain
1. vps: `dokku domains:enable APP_NAME`
2. vps: `dokku domains:add APP_NAME THE_DOMAIN_NAME`
3. change DNS to your VPS IP_ADDRESS
4. done
