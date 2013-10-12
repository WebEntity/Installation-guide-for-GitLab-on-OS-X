# Installation guide for GitLab 6.1 on OS X 10.8.5 with Server 2.2.2

## Requirements
- Mac OS X 10.8.5
- Server app
- User group `git` and user `git` in this group
- Enable remote login for `git` user
- Installed Xcode
- Installed command line tools in Xcode

Hide the git user from the login screen:

	 sudo defaults write /Library/Preferences/com.apple.loginwindow HiddenUsersList -array-add git

Unhide:

	sudo defaults delete /Library/Preferences/com.apple.loginwindow HiddenUsersList

## Recommendation

Use Parallels Deskop for first install.

## Note

If you find any issues, please let me know or send PR with fix ;-) Thank you!

## Install instructions

1. Install Homebrew
2. Install some prerequisites
3. Install mysql
4. Setup database
5. Install ruby
6. Install Gitlab Shell
7. Install GitLab
8. Check Installation
9. Setting up Gitlab with Apache
10. Automatic backups

### 1. Install Homebrew

	ruby -e "$(curl -fsSL https://raw.github.com/mxcl/homebrew/go)"
	brew doctor

### 2. Install some prerequisites

	brew install icu4c # Necessary for the charlock_holmes gem install later
	brew install git # Install git 1.8 or greater...

	brew install redis
	ln -sfv /usr/local/opt/redis/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.redis.plist

Make sure you have python 2.5+ (gitlab don’t support python 3.x)

Confirm python 2.5+

	python --version

GitLab looks for python2

	sudo ln -s /usr/bin/python /usr/bin/python2

Some more dependices

	sudo easy_install pip
	sudo pip install pygments

Install `docutils` http://sourceforge.net/projects/docutils/files/latest/download?source=files

	cd docutil
	sudo python setup.py install

### 3. Install mysql

	brew install mysql
	ln -sfv /usr/local/opt/mysql/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist

### 4. Setup database

Run `mysql_secure_installation` and set a reoot password, disallow remote root login, remove the test database, and reload the privelege tables.

	mysql_secure_installation

Now login in mysql

	mysql -u root -pPASSWORD_HERE

Create a new user for our gitlab setup 'gitlab'

	CREATE USER 'gitlab'@'localhost' IDENTIFIED BY 'PASSWORD_HERE';

Create database

	CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;

Grant the GitLab user necessary permissions on the table.

	GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'gitlab'@'localhost';

Quit the database session

	\q

Try connecting to the new database with the new user

	sudo -u git -H mysql -u gitlab -pPASSWORD_HERE -D gitlabhq_production

### 5. Install ruby

Install rbenv and ruby-build

	brew install rbenv
	brew install ruby-build

Make sure rbenv loads in the git user's shell

	echo 'export PATH="/usr/local/bin:$PATH"' | sudo -u git tee -a /Users/git/.profile
	echo 'if which rbenv > /dev/null; then eval "$(rbenv init -)"; fi' | sudo -u git tee -a /Users/git/.profile
	sudo -u git cp /Users/git/.profile /Users/git/.bashrc

If you get the following error on OS X 10.8.5 or lower:
`./bin/install:3: undefined method `require_relative' for main:Object (NoMethodError)`
Do the following to update to the proper Ruby version

	echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bash_profile
	echo 'eval "$(rbenv init - --no-rehash)"' >> ~/.bash_profile
	. ~/.bash_profile

Install ruby for the git user

	sudo -u git -H -i rbenv install 1.9.3-p448
	sudo -u git -H -i 'rbenv global 1.9.3-p448'

Install ruby for your user too

	rbenv install 1.9.3-p448
	rbenv global 1.9.3-p448

### 6. Install Gitlab Shell

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlab-shell.git
	cd gitlab-shell
	sudo -u git git checkout v1.7.1
	sudo -u git cp config.yml.example config.yml

Now open `config.yml` file and edit it

Set the `gitlab_url`. Replace gitlab.example.com wih your url (domain.com)

	sudo -u git sed -i "" "s/localhost/domain.com/" config.yml

Use `/Users` instead of `/home`, and change redis-cli path to homebrew’s redis-cli

	sudo -u git sed -i "" "s/\/home\//\/Users\//g" config.yml
	sudo -u git sed -i "" "s/\/usr\/bin\/redis-cli/\/usr\/local\/bin\/redis-cli/" config.yml

Do setup

	sudo -u git -H ./bin/install

### 7. Install Gitlab

#### Download Gitlab

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlabhq.git gitlab
	cd gitlab
	sudo -u git git checkout 6-1-stable

#### Configuring GitLab

	sudo -u git cp config/gitlab.yml.example config/gitlab.yml
	sudo -u git sed -i "" "s/\/usr\/bin\/git/\/usr\/local\/bin\/git/g" config/gitlab.yml
	sudo -u git sed -i "" "s/\/home/\/Users/g" config/gitlab.yml
	sudo -u git sed -i "" "s/localhost/domain.com/g" config/gitlab.yml

Make sure GitLab can write to the `log/` and `tmp/` directories

	sudo chown -R git log/
	sudo chown -R git tmp/
	sudo chmod -R u+rwX  log/
	sudo chmod -R u+rwX  tmp/

Create directories for repositories make sure GitLab can write to them

	sudo -u git mkdir /Users/git/repositories
	sudo chmod -R u+rwX  /Users/git/repositories/

Create directory for satellites

	sudo -u git mkdir /Users/git/gitlab-satellites

Create directories for sockets/pids and make sure GitLab can write to them

	sudo -u git mkdir tmp/pids/
	sudo -u git mkdir tmp/sockets/

	sudo chmod -R u+rwX  tmp/pids/
	sudo chmod -R u+rwX  tmp/sockets/

Create public/uploads directory otherwise backup will fail

	sudo -u git mkdir public/uploads
	sudo chmod -R u+rwX  public/uploads

Fix

	sudo chown -R git:git /Users/git/repositories/
	sudo chmod -R ug+rwX,o-rwx /Users/git/repositories/
	sudo chmod -R ug-s /Users/git/repositories/
	sudo find /Users/git/repositories/ -type d -print0 | sudo xargs -0 chmod g+s

Copy the example Unicorn config

	sudo -u git cp config/unicorn.rb.example config/unicorn.rb
	sudo -u git sed -i "" "s/\/home/\/Users/g" config/unicorn.rb

Change `listen` to `listen "127.0.0.1:8080"` and comment out gitlab.socket in `unicorn.rb`.

Configure Git global settings for git user, useful when editing via web

	sudo -u git -H git config --global user.name "GitLab"
	sudo -u git -H git config --global user.email "gitlab@domain.com"

#### Gitlab Mysql Config

	sudo -u git cp config/database.yml.mysql config/database.yml
	sudo -u git sed -i "" "s/root/gitlab/g" config/database.yml
	sudo -u git sed -i "" "s/secure password/PASSWORD_HERE/g" config/database.yml

#### Install Gems

	sudo gem install charlock_holmes --version '0.6.9.4'
	sudo -u git -H bash -l -c 'gem install bundler'
	sudo -u git -H bash -l -c 'rbenv rehash'
	sudo -u git -H bash -l -c 'bundle install --deployment --without development test postgres'

For error with `mysql2` gem you need to edit `mysql_config` file in `/usr/local/Cellar/mysql/5.6.12/bin` and removed the W-compiler options `-Wno-null-conversion` and `-Wno-unused-private-field` for `cflags` and `cxxflags`.

#### Initialize Database and Activate Advanced Features

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:setup RAILS_ENV=production'

Here is your admin login credentials:

	login: admin@local.host
	password: 5iveL!fe

#### Install Init script

	sudo mkdir /etc/init.d
	sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
	sudo chmod +x /etc/init.d/gitlab
	sudo sed -i "" "s/\/home\//\/Users\//g" /etc/init.d/gitlab
	sudo /etc/init.d/gitlab start

### 8. Check Installation

Check gitlab-shell

	sudo -u git /Users/git/gitlab-shell/bin/check

Double-check environment configuration

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:env:info RAILS_ENV=production'

Do a thorough check. Make sure everything is green.

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:check RAILS_ENV=production'

The script complained about the init script not being up-to-date, but I assume that’s because it was modified to use /Users instead of /home. You can safely ignore that warning.

### 9. Setting up Gitlab with Apache

1. Setup website in Server.app (with SSL)
2. Go to `/Library/Server/Web/Config/apache2/sites`
3. Stop the webserver
4. Edit your site config, for example `0000_any_443_domain.com.conf`, like vhost config in this repo.
5. Start the webserver

### 10. Automatic backups

Copy `com.webentity.gitlab_backup.plist` to `/Library/LaunchDaemons/` and setup it.

	sudo cp com.webentity.gitlab_backup.plist /Library/LaunchDaemons/
	sudo launchctl load /Library/LaunchDaemons/com.webentity.gitlab_backup.plist

I recomend to uncomment `keep_time` in `gitlab.yml` Backup settings.

## ToDo

- LaunchDaemon for GitLab (`com.webentity.gitlab.plist` in this repo does not work - maybe `/etc/init.d/gitlab` need some tweaks)

## Links and sources ( thank you guys :-) )

- http://www.makebetterthings.com/git/install-gitlab-5-3-on-mac-os-x-server-10-8-4/
- http://thoughtpointers.net/2013/05/23/installing-gitlab-v52-on-os-x/
- http://createdbypete.com/articles/installing-gitlab-on-mac-os-x-and-mac-os-x-server/
- https://gist.github.com/jasoncodes/1223731
- http://stackoverflow.com/questions/11945425/unable-to-install-mysql2-gem-os-x-mountain-lion
- http://stackoverflow.com/questions/3754662/errors-installing-mysql2-gem-via-the-bundler
- http://ryanbigg.com/2011/06/mac-os-x-ruby-rvm-rails-and-you/
- https://groups.google.com/forum/#!topic/gitlabhq/dPUn77dxNSQ
- https://gist.github.com/carlosjrcabello/5486422
