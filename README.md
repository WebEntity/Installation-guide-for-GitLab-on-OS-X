# Installation guide for GitLab 8.8 on OS X 10.11

> This is WIP version for OS X 10.11. For OS X 10.10 see [10.10 branch](https://github.com/WebEntity/Installation-guide-for-GitLab-on-OS-X/tree/10.10).

## Overview

The GitLab installation consists of setting up the following components:

1.  Packages / Dependencies
2.  System User
3.  Ruby
4.  Go
5.  Database
6.  Redis
7.  GitLab
8.  Nginx

## 1. Packages / Dependencies

Command line tools

```
xcode-select --install #xcode command line tools
```

Homebrew

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install icu4c git logrotate libxml2 cmake pkg-config openssl
brew link openssl --force
```

Make sure you have python 2.5+ (gitlab don’t support python 3.x)

Confirm python 2.5+

```
python --version
```

GitLab looks for python2

```
sudo ln -s /usr/bin/python /usr/bin/python2
```

> On OS X 10.11 it won't work. You need to disable [SIP](https://en.wikipedia.org/wiki/System_Integrity_Protection).

Some more dependices

```
sudo easy_install pip
sudo pip install pygments
```

Install `docutils` from [source](http://sourceforge.net/projects/docutils/files/latest/download?source=files).

```
curl -O http://heanet.dl.sourceforge.net/project/docutils/docutils/0.12/docutils-0.12.tar.gz
gunzip -c docutils-0.12.tar.gz | tar xopf -
cd docutils-0.12
sudo python setup.py install
```

## 2. System User

Run the following commands in order to create the group and user `git`:

```bash
LastUserID=$(dscl . -list /Users UniqueID | awk '{print $2}' | sort -n | tail -1)
NextUserID=$((LastUserID + 1))
sudo dscl . create /Users/git
sudo dscl . create /Users/git RealName "GitLab"
sudo dscl . create /Users/git hint "Password Hint"
sudo dscl . create /Users/git UniqueID $NextUserID
LastGroupID=$(dscl . readall /Groups | grep PrimaryGroupID | awk '{ print $2 }' | sort -n | tail -1)
NextGroupID=$(($LastGroupID + 1 ))
sudo dscl . create /Groups/git
sudo dscl . create /Groups/git RealName "GitLab"
sudo dscl . create /Groups/git passwd "*"
sudo dscl . create /Groups/git gid $NextGroupID
sudo dscl . create /Users/git PrimaryGroupID $NextGroupID
sudo dscl . create /Users/git UserShell $(which bash)
sudo dscl . create /Users/git NFSHomeDirectory /Users/git
sudo cp -R /System/Library/User\ Template/English.lproj /Users/git
sudo chown -R git:git /Users/git
```

Hide the git user from the login screen:

```
sudo defaults write /Library/Preferences/com.apple.loginwindow HiddenUsersList -array-add git
```

Unhide:

```
sudo defaults delete /Library/Preferences/com.apple.loginwindow HiddenUsersList
```

## 3. Ruby

> The use of Ruby version managers such as [RVM](http://rvm.io/), [rbenv](https://github.com/sstephenson/rbenv) or [chruby](https://github.com/postmodern/chruby) with GitLab in production frequently leads to hard to diagnose problems. For example, GitLab Shell is called from OpenSSH and having a version manager can prevent pushing and pulling over SSH. Version managers are not supported and we strongly advise everyone to follow the instructions below to use a system Ruby.

On OS X we are forced to use non-system ruby and install it using version manager.

Install rbenv and ruby-build

```
brew install rbenv ruby-build
```

Make sure rbenv loads in the git user's shell

```
echo 'export PATH="/usr/local/bin:$PATH"' | sudo -u git tee -a /Users/git/.profile
echo 'if which rbenv > /dev/null; then eval "$(rbenv init -)"; fi' | sudo -u git tee -a /Users/git/.profile
sudo -u git cp /Users/git/.profile /Users/git/.bashrc
```

If you get the following error on OS X 10.8.5 or lower:
`./bin/install:3: undefined method `require_relative' for main:Object (NoMethodError)`
Do the following to update to the proper Ruby version

```
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init - --no-rehash)"' >> ~/.bash_profile
. ~/.bash_profile
```

Install ruby for the git user

```
sudo -u git -H -i 'rbenv install 2.1.8'
sudo -u git -H -i 'rbenv global 2.1.8'
```

Install ruby for your user too (optional)

```
rbenv install 2.1.8
rbenv global 2.1.8
```

## 4. Go

Since GitLab 8.0, Git HTTP requests are handled by gitlab-git-http-server.
This is a small daemon written in Go.
To install gitlab-git-http-server we need a Go compiler.

```
brew install go
```

## 5. Database

Gitlab recommends using a PostgreSQL database. But you can use MySQL too, see [MySQL setup guide](database_mysql.md).

```
brew install postgresql
ln -sfv /usr/local/opt/postgresql/*.plist ~/Library/LaunchAgents
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist
```

Login to PostgreSQL

```
psql -d postgres
```

Create a user for GitLab.

```
CREATE USER git;
```

Create the GitLab production database & grant all privileges on database

```
CREATE DATABASE gitlabhq_production OWNER git;
```

Quit the database session

```
\q
```

Try connecting to the new database with the new user

```
sudo -u git -H psql -d gitlabhq_production
```

## 6. Redis

```
brew install redis
ln -sfv /usr/local/opt/redis/*.plist ~/Library/LaunchAgents
```

Redis config is located in `/usr/local/etc/redis.conf`. Make a copy:

```
cp /usr/local/etc/redis.conf /usr/local/etc/redis.conf.orig
```

Disable Redis listening on TCP by setting 'port' to 0

```
sed 's/^port .*/port 0/' /usr/local/etc/redis.conf.orig | sudo tee /usr/local/etc/redis.conf
```

Edit file (`nano /usr/local/etc/redis.conf`) and uncomment:

```
unixsocket /tmp/redis.sock
unixsocketperm 777
```

Start Redis

```
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.redis.plist
```

## 7. GitLab

```
cd /Users/git
```

### Clone the Source

Clone GitLab repository

```
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 8-8-stable gitlab
```

**Note:** You can change `8-8-stable` to `master` if you want the *bleeding edge* version, but never install master on a production server!

### Configure It

Go to GitLab installation folder

```
cd /Users/git/gitlab
```

Copy the example GitLab config

```
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
sudo -u git sed -i "" "s/\/usr\/bin\/git/\/usr\/local\/bin\/git/g" config/gitlab.yml
sudo -u git sed -i "" "s/\/home/\/Users/g" config/gitlab.yml
```

Update GitLab config file, follow the directions at top of file

```
sudo -u git -H nano config/gitlab.yml
```

Copy the example secrets file

```
sudo -u git -H cp config/secrets.yml.example config/secrets.yml
sudo -u git -H chmod 0600 config/secrets.yml
```

Make sure GitLab can write to the log/ and tmp/ directories

```
sudo chown -R git log/
sudo chown -R git tmp/
sudo chmod -R u+rwX,go-w log/
sudo chmod -R u+rwX tmp/
```

Make sure GitLab can write to the tmp/pids/ and tmp/sockets/ directories

```
sudo chmod -R u+rwX tmp/pids/
sudo chmod -R u+rwX tmp/sockets/
```

Make sure GitLab can write to the public/uploads/ directory

```
sudo chmod 0700 public/uploads
```

Make sure GitLab can write to the repositories directory

```
sudo chmod -R ug+rwX,o-rwx /Users/git/repositories/
sudo chmod -R ug-s /Users/git/repositories/
sudo find /Users/git/repositories/ -type d -print0 | sudo xargs -0 chmod g+s
```

Change the permissions of the directory where CI build traces are stored

```
sudo chmod -R u+rwX builds/
```

Copy the example Unicorn config

```
sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
sudo -u git sed -i "" "s/\/home/\/Users/g" config/unicorn.rb
```

Find number of cores

```
sysctl -n hw.ncpu
```

Enable cluster mode if you expect to have a high load instance
Ex. change amount of workers to 3 for 2GB RAM server
Set the number of workers to at least the number of cores

```
sudo -u git -H nano config/unicorn.rb
```

Copy the example Rack attack config

```
sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb
```

Configure Git global settings for git user, used when editing via web editor

```
sudo -u git -H git config --global core.autocrlf input
```

Disable `git gc --auto` because GitLab runs `git gc` for us already.

```
sudo -u git -H git config --global gc.auto 0
```

Configure Redis connection settings

```
sudo -u git -H cp config/resque.yml.example config/resque.yml
```

Change the Redis socket path to `/tmp/redis.sock`:

```
sudo -u git -H nano config/resque.yml
```

**Important Note:** Make sure to edit both `gitlab.yml` and `unicorn.rb` to match your setup.

**Note:** If you want to use HTTPS, see [Using HTTPS](#using-https) for the additional steps.

### Configure GitLab DB Settings

PostgreSQL only:

```
sudo -u git cp config/database.yml.postgresql config/database.yml
```

MySQL only:

```
sudo -u git cp config/database.yml.mysql config/database.yml
```

MySQL and remote PostgreSQL only:
Update username/password in config/database.yml.
You only need to adapt the production settings (first part).
If you followed the database guide then please do as follows:
Change 'secure password' with the value you have given to $password
You can keep the double quotes around the password

```
sudo -u git -H nano config/database.yml
```

PostgreSQL and MySQL:
Make config/database.yml readable to git only

```
sudo -u git -H chmod o-rwx config/database.yml
```

### Install Gems

**Note:** As of bundler 1.5.2, you can invoke `bundle install -jN` (where `N` the number of your processor cores) and enjoy the parallel gems installation with measurable difference in completion time (~60% faster). Check the number of your cores with `nproc`. For more information check this [post](http://robots.thoughtbot.com/parallel-gem-installing-using-bundler). First make sure you have bundler >= 1.5.2 (run `bundle -v`) as it addresses some [issues](https://devcenter.heroku.com/changelog-items/411) that were [fixed](https://github.com/bundler/bundler/pull/2817) in 1.5.2.

Preparation:

```
sudo su git
. ~/.profile
gem install bundler --no-ri --no-rdoc
rbenv rehash
cd ~/gitlab/
```

For PostgreSQL (note, the option says "without ... mysql")

```
bundle install --deployment --without development test mysql aws kerberos
```

Or if you use MySQL (note, the option says "without ... postgres")

```
bundle install --deployment --without development test postgres aws kerberos
```

**Note:** If you want to use Kerberos for user authentication, then omit `kerberos` in the `--without` option above.

### Install GitLab Shell

GitLab Shell is an SSH access and repository management software developed specially for GitLab.

Run the installation task for gitlab-shell (replace `REDIS_URL` if needed):

```
sudo su git
. ~/.profile
cd ~/gitlab/
bundle exec rake gitlab:shell:install[v2.7.2] REDIS_URL=unix:/tmp/redis.sock RAILS_ENV=production
```

By default, the gitlab-shell config is generated from your main GitLab config.
You can review (and modify) the gitlab-shell config as follows:

```
sudo -u git -H nano /Users/git/gitlab-shell/config.yml
```

**Note:** If you want to use HTTPS, see [Using HTTPS](#using-https) for the additional steps.

**Note:** Make sure your hostname can be resolved on the machine itself by either a proper DNS record or an additional line in /etc/hosts ("127.0.0.1  hostname"). This might be necessary for example if you set up gitlab behind a reverse proxy. If the hostname cannot be resolved, the final installation check will fail with "Check GitLab API access: FAILED. code: 401" and pushing commits will be rejected with "[remote rejected] master -> master (hook declined)".

### Install gitlab-workhorse

```
cd /Users/git
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-workhorse.git
cd gitlab-workhorse
sudo -u git -H git checkout 0.7.1
sudo -u git -H make
```

### Initialize Database and Activate Advanced Features

```
sudo su git
. ~/.profile
cd ~/gitlab/
bundle exec rake gitlab:setup RAILS_ENV=production
```

Type 'yes' to create the database tables.
When done you see 'Administrator account created:

**Note:** You can set the Administrator/root password by supplying it in environmental variable `GITLAB_ROOT_PASSWORD` as seen below. If you don't set the password (and it is set to the default one) please wait with exposing GitLab to the public internet until the installation is done and you've logged into the server the first time. During the first login you'll be forced to change the default password.

```
bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD=yourpassword
```

### Secure secrets.yml

The `secrets.yml` file stores encryption keys for sessions and secure variables.
Backup `secrets.yml` someplace safe, but don't store it in the same place as your database backups.
Otherwise your secrets are exposed if one of your backups is compromised.

### Install Init Script

Download the init script (will be `/etc/init.d/gitlab`):

```
cd /Users/git/gitlab
sudo mkdir -p /etc/init.d/
sudo mkdir -p /etc/default/
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
```

Since you are installing to a folder other than default ```/home/users/git/gitlab```, copy and edit the defaults file:

```
curl -O https://raw.githubusercontent.com/WebEntity/Installation-guide-for-GitLab-on-OS-X/master/gitlab.default.osx
sudo cp gitlab.default.osx /etc/default/gitlab.default
```

If you installed GitLab in another directory or as a user other than the default you should change these settings in `/etc/default/gitlab`. Do not edit `/etc/init.d/gitlab` as it will be changed on upgrade.


### Setup Logrotate
```
sudo cp lib/support/logrotate/gitlab /usr/local/etc/logrotate.d/gitlab
sudo sed -i "" "s/\/home/\/Users/g" /usr/local/etc/logrotate.d/gitlab
ln -sfv /usr/local/opt/logrotate/*.plist ~/Library/LaunchAgents
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.logrotate.plist
```

### Check Application Status

Check if GitLab and its environment are configured correctly:
```
sudo su git
. ~/.profile
cd ~/gitlab/
bundle exec rake gitlab:env:info RAILS_ENV=production
```

### Compile Assets
```
sudo su git
. ~/.profile
cd ~/gitlab/
bundle exec rake assets:precompile RAILS_ENV=production
```

### Start Your GitLab Instance
```
sudo sh /etc/init.d/gitlab start
```

## 8. Nginx

**Note:** Nginx is the officially supported web server for GitLab. If you cannot or do not want to use Nginx as your web server, have a look at the [GitLab recipes](https://gitlab.com/gitlab-org/gitlab-recipes/).

### Installation
```
brew install nginx
sudo mkdir -p /var/log/nginx/
```

### Site Configuration

Default nginx configuration has an example server on port 8080, same as Gitlab Unicorn instance, which will collide and Gitlab won't start.
Edit nginx configuration and comment out whole example server block for it to work together:

```
sudo nano /usr/local/etc/nginx/nginx.conf
```

Copy the example site config:

```
sudo cp lib/support/nginx/gitlab /usr/local/etc/nginx/servers/gitlab
sudo sed -i "" "s/\/home/\/Users/g" /usr/local/etc/nginx/servers/gitlab
```

Make sure to edit the config file to match your setup:

Change YOUR_SERVER_FQDN to the fully-qualified domain name of your host serving GitLab.

```
sudo nano /usr/local/etc/nginx/servers/gitlab
```

**Note:** If you want to use HTTPS, replace the `gitlab` Nginx config with `gitlab-ssl`. See [Using HTTPS](#using-https) for HTTPS configuration details.

### Test Configuration

Validate your `gitlab` or `gitlab-ssl` Nginx config file with the following command:

```
sudo nginx -t
```

You should receive `syntax is okay` and `test is successful` messages. If you receive errors check your `gitlab` or `gitlab-ssl` Nginx config file for typos, etc. as indicated in the error message given.

### Start

```
sudo nginx
```

## Done!

### Double-check Application Status

To make sure you didn't miss anything run a more thorough check with:

```
sudo su git
. ~/.profile
cd ~/gitlab/
bundle exec rake gitlab:check RAILS_ENV=production
```

If all items are green, then congratulations on successfully installing GitLab!

NOTE: Supply `SANITIZE=true` environment variable to `gitlab:check` to omit project names from the output of the check command.

### Initial Login

Visit YOUR_SERVER in your web browser for your first GitLab login. The setup has created a default admin account for you. You can use it to log in:

```
root
5iveL!fe
```

**Important Note:** On login you'll be prompted to change the password.

**Enjoy!**

You can use `sudo sh /etc/init.d/gitlab start`, `sudo sh /etc/init.d/gitlab stop` and `sudo sh /etc/init.d/gitlab restart` to manually start, stop and restart GitLab.

### Autostart on boot

Copy Nginx and Gitlab plists and load it:

```
sudo cp /usr/local/opt/nginx/homebrew.mxcl.nginx.plist /Library/LaunchDaemons/
sudo cp com.webentity.gitlab.plist /Library/LaunchDaemons/
sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.nginx.plist
sudo launchctl load /Library/LaunchDaemons/com.webentity.gitlab.plist
```

## Advanced Setup Tips

### Using HTTPS

To use GitLab with HTTPS:

1. In `gitlab.yml`:
    1. Set the `port` option in section 1 to `443`.
    1. Set the `https` option in section 1 to `true`.
1. In the `config.yml` of gitlab-shell:
    1. Set `gitlab_url` option to the HTTPS endpoint of GitLab (e.g. `https://git.example.com`).
    1. Set the certificates using either the `ca_file` or `ca_path` option.
1. Use the `gitlab-ssl` Nginx example config instead of the `gitlab` config.
    1. Update `YOUR_SERVER_FQDN`.
    1. Update `ssl_certificate` and `ssl_certificate_key`.
    1. Review the configuration file and consider applying other security and performance enhancing features.

Using a self-signed certificate is discouraged but if you must use it follow the normal directions then:

1. Generate a self-signed SSL certificate:

    ```
    mkdir -p /etc/nginx/ssl/
    cd /etc/nginx/ssl/
    sudo openssl req -newkey rsa:2048 -x509 -nodes -days 3560 -out gitlab.crt -keyout gitlab.key
    sudo chmod o-r gitlab.key
    ```

1. In the `config.yml` of gitlab-shell set `self_signed_cert` to `true`.

### More

You can find more tips in [official documentation](https://github.com/gitlabhq/gitlabhq/blob/8-0-stable/doc/install/installation.md#advanced-setup-tips).

## Todo

- Working LaunchDeamon for Gitlab backups
