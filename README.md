# Installation guide for GitLab 6.7 on OS X 10.9 with Server 3

## Requirements
- Mac OS X 10.9
- Server 3
- User group `git` and user `git` in this group
- Enable remote login for `git` user

Hide the git user from the login screen:

	 sudo defaults write /Library/Preferences/com.apple.loginwindow HiddenUsersList -array-add git

Unhide:

	sudo defaults delete /Library/Preferences/com.apple.loginwindow HiddenUsersList

## Recommendation

Use Parallels Deskop for first install.

## Note

If you find any issues, please let me know or send PR with fix ;-) Thank you!

## Install instructions

1. [Install command line tools](#1-install-command-line-tools)
2. [Install Homebrew](#2-install-homebrew)
3. [Install some prerequisites](#3-install-some-prerequisites)
4. [Install mysql](#4-install-mysql)
5. [Setup database](#5-setup-database)
6. [Install ruby](#6-install-ruby)
7. [Install Gitlab Shell](#7-install-gitlab-shell)
8. [Install GitLab](#8-install-gitlab)
9. [Check Installation](#9-check-installation)
10. [Setting up Gitlab with Apache](#10-setting-up-gitlab-with-apache)
11. [Automatic backups](#11-automatic-backups)
12. [Configuring SMTP](#12-configuring-smtp)

### 1. Install command line tools

	xcode-select --install #xcode command line tools

### 2. Install Homebrew

	ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"
	brew doctor

### 3. Install some prerequisites
	
	brew install icu4c git logrotate redis libxml2

	ln -sfv /usr/local/opt/logrotate/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.logrotate.plist
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

Install `docutils` from http://sourceforge.net/projects/docutils/files/latest/download?source=files

	curl -O http://heanet.dl.sourceforge.net/project/docutils/docutils/0.11/docutils-0.11.tar.gz
	gunzip -c docutils-0.11.tar.gz | tar xopf -
	cd docutils-0.11
	sudo python setup.py install

### 4. Install database (postgresql or mysql)

> Official installation documentation recommend to use postgresql, see [http://doc.gitlab.com/ce/install/installation.html](http://doc.gitlab.com/ce/install/installation.html).

#### postgresql
	brew install postgresql
	ln -sfv /usr/local/opt/postgresql/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist

#### mysql

	brew install mysql
	ln -sfv /usr/local/opt/mysql/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist

### 5. Setup database

#### postgresql

Connect to postgres database

	psql postgres

> If this operation gives you an error `psql: could not connect to server: No such file or directory` check that you are using psql from homebrew (not the one which is installed with Mac OS X). You can find all of the installed psql with `which -a psql`. To fix that you can use fully qualified path to psql or just fix `$PATH` variable by placing path to homebrew `bin` before others. 

Login to PostgreSQL
	psql -d postgres

Create a user for GitLab.
	postgres=# CREATE USER git;

Create the GitLab production database & grant all privileges on database
	postgres=# CREATE DATABASE gitlabhq_production OWNER git;

Quit the database session
	postgres=# \q

Try connecting to the new database with the new user
	sudo -u git -H psql -d gitlabhq_production

#### mysql

Run `mysql_secure_installation` and set a root password, disallow remote root login, remove the test database, and reload the privelege tables.

	mysql_secure_installation

Now login in mysql

	mysql -u root -pPASSWORD_HERE

Create a new user for our gitlab setup 'git'

	CREATE USER 'git'@'localhost' IDENTIFIED BY 'PASSWORD_HERE';

Create database

	CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;

Grant the GitLab user necessary permissions on the table.

	GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'git'@'localhost';

Quit the database session

	\q

Try connecting to the new database with the new user

	sudo -u git -H mysql -u git -pPASSWORD_HERE -D gitlabhq_production

### 6. Install ruby

OS X 10.9 has ruby 2.0. No need to install anything.

### 7. Install Gitlab Shell

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlab-shell.git
	cd gitlab-shell
	sudo -u git git checkout v1.9.1
	sudo -u git cp config.yml.example config.yml

Now open `config.yml` file and edit it

Set the `gitlab_url`. Replace gitlab.example.com wih your url (domain.com)

	sudo -u git sed -i "" "s/localhost/domain.com/" config.yml

Use `/Users` instead of `/home`, and change redis-cli path to homebrew’s redis-cli

	sudo -u git sed -i "" "s/\/home\//\/Users\//g" config.yml
	sudo -u git sed -i "" "s/\/usr\/bin\/redis-cli/\/usr\/local\/bin\/redis-cli/" config.yml

Do setup

	sudo -u git -H ./bin/install

### 8. Install Gitlab

#### Download Gitlab

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlabhq.git gitlab
	cd gitlab
	sudo -u git git checkout 6-7-stable

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

Comment out `listen "/Users/git/gitlab/tmp/sockets/gitlab.socket", :backlog => 64` in `unicorn.rb`.

Configure Git global settings for git user, useful when editing via web

	sudo -u git -H git config --global user.name "GitLab"
	sudo -u git -H git config --global user.email "gitlab@domain.com"

Copy rack attack middleware config

	sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

Set up logrotate
	
	sudo mkdir /etc/logrotate.d/
	sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
	sudo sed -i "" "s/\/home/\/Users/g" /etc/logrotate.d/gitlab

#### Gitlab Database Config

##### postgresql

> By default homebrew installs postgresql with allowing access to it with local accounts, so no needs of changing passwords.

	sudo -u git cp config/database.yml.postgresql config/database.yml

##### mysql

	sudo -u git cp config/database.yml.mysql config/database.yml
	sudo -u git sed -i "" "s/secure password/PASSWORD_HERE/g" config/database.yml

#### Install Gems

You need to edit `Gemfile.lock` (`sudo -u git nano Gemfile.lock`) and change the versions of `underscore-rails` to `1.5.2` (in two places). You also need to edit `Gemfile` (`sudo -u git nano Gemfile`) to change `underscore-rails` to `1.5.2`.

In case if you are using postgres as database

	sudo gem install bundler
	sudo bundle install --deployment --without development test mysql aws
	
In case if you are using mysql as database

	sudo gem install bundler
	sudo bundle install --deployment --without development test postgres aws

If you see error with `version_sorter` gem run this:

If you are using postgres
	sudo ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future bundle install --deployment --without development test mysql aws

If you are using mysql

	sudo ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future bundle install --deployment --without development test postgres aws

#### Initialize Database and Activate Advanced Features

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:setup RAILS_ENV=production'

Here is your admin login credentials:

	login: admin@local.host
	password: 5iveL!fe

#### Install web and background_jobs services

Next step will setup services which will keep Gitlab up and running

	sudo curl --output /Library/LaunchDaemons/gitlab.web.plist https://raw.githubusercontent.com/CiTroNaK/Installation-guide-for-GitLab-on-OS-X/master/gitlab.web.plist
	sudo launchctl load /Library/LaunchDaemons/gitlab.web.plist

	sudo curl --output /Library/LaunchDaemons/gitlab.background_jobs.plist https://raw.githubusercontent.com/CiTroNaK/Installation-guide-for-GitLab-on-OS-X/master/gitlab.background_jobs.plist
	sudo launchctl load /Library/LaunchDaemons/gitlab.background_jobs.plist

> `ProgramArguments` arrays in these plists should be in sync with `start` functions in scripts [background_jobs](https://github.com/gitlabhq/gitlabhq/blob/master/script/background_jobs) and [web](https://github.com/gitlabhq/gitlabhq/blob/master/script/web). 

### 9. Check Installation

Check gitlab-shell

	sudo -u git /Users/git/gitlab-shell/bin/check

Double-check environment configuration

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:env:info RAILS_ENV=production'

Do a thorough check. Make sure everything is green.

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:check RAILS_ENV=production'

The script complained about the init script not being up-to-date, but I assume that’s because it was modified to use /Users instead of /home. You can safely ignore that warning.

### 10. Setting up Gitlab with Apache

1. Setup website in Server.app (with SSL)
2. Go to `/Library/Server/Web/Config/apache2/sites`
3. Stop the webserver
4. Edit your site config, for example `0000_any_443_domain.com.conf`, like vhost config in this repo.
5. Start the webserver

### 11. Automatic backups

Copy `gitlab.backup.plist` to `/Library/LaunchDaemons/` and setup it.

	sudo curl --output /Library/LaunchDaemons/gitlab.backup.plist https://raw.githubusercontent.com/CiTroNaK/Installation-guide-for-GitLab-on-OS-X/master/gitlab.backup.plist
	sudo launchctl load /Library/LaunchDaemons/gitlab.backup.plist

I recommend to uncomment `keep_time` in `gitlab.yml` Backup settings.

> You can verify backup service with command `sudo launchctl start gitlab.backup` and by looking in logs under `\Users\git\gitlab\log\backup.stderr.log` and `\Users\git\gitlab\log\backup.stdout.log`.

### 12. Configuring SMTP

Copy config file

	sudo -u git -H cp config/initializers/smtp_settings.rb.sample config/initializers/smtp_settings.rb

Edit `config/initializers/smtp_settings.rb` with your settings (see [ActionMailer::Base - Configuration options](http://api.rubyonrails.org/classes/ActionMailer/Base.html))

## Links and sources ( thank you guys :-) )

- http://stackoverflow.com/questions/22567971/ruby-gem-version-sorter-on-os-x-10-9
- http://www.makebetterthings.com/git/install-gitlab-5-3-on-mac-os-x-server-10-8-4/
- http://thoughtpointers.net/2013/05/23/installing-gitlab-v52-on-os-x/
- http://createdbypete.com/articles/installing-gitlab-on-mac-os-x-and-mac-os-x-server/
- https://gist.github.com/jasoncodes/1223731
- http://stackoverflow.com/questions/11945425/unable-to-install-mysql2-gem-os-x-mountain-lion
- http://stackoverflow.com/questions/3754662/errors-installing-mysql2-gem-via-the-bundler
- http://ryanbigg.com/2011/06/mac-os-x-ruby-rvm-rails-and-you/
- https://groups.google.com/forum/#!topic/gitlabhq/dPUn77dxNSQ
- https://gist.github.com/carlosjrcabello/5486422


[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/CiTroNaK/installation-guide-for-gitlab-on-os-x/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

