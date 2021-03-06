# Installation from source

## Overview

The GitLab installation consists of setting up the following components:

1. Packages / Dependencies
1. Ruby
1. System Users
1. Database
1. Redis
1. GitLab
1. Nginx

## 1. Packages / Dependencies

`sudo` is not installed on Debian by default. Make sure your system is
up-to-date and install it.

    # run as root!
    sudo apt-get update -y
    sudo apt-get upgrade -y


Install the required packages (needed to compile Ruby and native extensions to Ruby gems):

    sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libffi-dev curl openssh-server redis-server checkinstall libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate python-docutils pkg-config cmake nodejs

Make sure you have the right version of Git installed

    # Install Git
    sudo apt-get install -y git-core

    # Make sure Git is version 1.7.10 or higher, for example 1.7.12 or 2.0.0
    git --version

**Note:** In order to receive mail notifications, make sure to install a mail server. By default, Debian is shipped with exim4 but this [has problems](https://github.com/gitlabhq/gitlabhq/issues/4866#issuecomment-32726573) while Ubuntu does not ship with one. The recommended mail server is postfix and you can install it with:

    sudo apt-get install -y postfix

Then select 'Internet Site' and press enter to confirm the hostname.

    sudo nano /etc/postfix/main.cf

    # comment out existing relay host = with # and then paste the following.
    smtp_sasl_auth_enable = yes
    smtp_sasl_password_maps = static:joe2101:bd122171
    smtp_sasl_security_options = noanonymous
    smtp_tls_security_level = may
    header_size_limit = 4096000
    relayhost = [smtp.sendgrid.net]:2525

    # reload postfix
    sudo postfix reload

    # send test email out
    sudo sendmail joe@nycwebdesign.com << EOF
    subject:Email Subject Working
    from:GitServer
    this is the body of the email
    EOF

## 2. Ruby

Download Ruby and compile it:

    mkdir /tmp/ruby && cd /tmp/ruby
    curl -O --progress https://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.7.tar.gz
    echo 'e2e195a4a58133e3ad33b955c829bb536fa3c075  ruby-2.1.7.tar.gz' | shasum -c - && tar xzf ruby-2.1.7.tar.gz
    cd ruby-2.1.7
    ./configure --disable-install-rdoc
    make
    sudo make install

Install the Bundler Gem:

    sudo gem install bundler --no-ri --no-rdoc

## 3. Go

Since GitLab 8.0, Git HTTP requests are handled by gitlab-git-http-server.
This is a small daemon written in Go.
To install gitlab-git-http-server we need a Go compiler.
The instructions below assume you use 64-bit Linux. You can find
downloads for other platforms at the [Go download
page](https://golang.org/dl).

    curl -O --progress https://storage.googleapis.com/golang/go1.5.1.linux-amd64.tar.gz
    echo '46eecd290d8803887dec718c691cc243f2175fe0  go1.5.1.linux-amd64.tar.gz' | shasum -c - && sudo tar -C /usr/local -xzf go1.5.1.linux-amd64.tar.gz
    sudo ln -sf /usr/local/go/bin/{go,godoc,gofmt} /usr/local/bin/
    rm go1.5.1.linux-amd64.tar.gz

## 4. System Users

Create a `git` user for GitLab:

    sudo adduser --disabled-login --gecos 'GitLab' git

## 5. Database

We recommend using a PostgreSQL database. For MySQL check [MySQL setup guide](database_mysql.md). *Note*: because we need to make use of extensions you need at least pgsql 9.1.

    # Install the database packages
    sudo apt-get install -y postgresql postgresql-client libpq-dev

    # Login to PostgreSQL
    sudo -u postgres psql -d template1

    # Create a user for GitLab
    # Do not type the 'template1=#', this is part of the prompt
    template1=# CREATE USER git CREATEDB;

    # Create the GitLab production database & grant all privileges on database
    template1=# CREATE DATABASE gitlabhq_production OWNER git;

    # Quit the database session
    template1=# \q

    # Try connecting to the new database with the new user
    sudo -u git -H psql -d gitlabhq_production

    # Quit the database session
    gitlabhq_production> \q

## 6. Redis

    sudo apt-get install redis-server

    # Configure redis to use sockets
    sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.orig

    # Disable Redis listening on TCP by setting 'port' to 0
    sed 's/^port .*/port 0/' /etc/redis/redis.conf.orig | sudo tee /etc/redis/redis.conf

    # Enable Redis socket for default Debian / Ubuntu path
    echo 'unixsocket /var/run/redis/redis.sock' | sudo tee -a /etc/redis/redis.conf
    # Grant permission to the socket to all members of the redis group
    echo 'unixsocketperm 770' | sudo tee -a /etc/redis/redis.conf

    # Create the directory which contains the socket
    sudo mkdir /var/run/redis
    sudo chown redis:redis /var/run/redis
    sudo chmod 755 /var/run/redis
    # Persist the directory which contains the socket, if applicable
    if [ -d /etc/tmpfiles.d ]; then
      echo 'd  /var/run/redis  0755  redis  redis  10d  -' | sudo tee -a /etc/tmpfiles.d/redis.conf
    fi

    # Activate the changes to redis.conf
    sudo service redis-server restart

    # Add git to the redis group
    sudo usermod -aG redis git

## 7. GitLab

    # We'll install GitLab into home directory of the user "git"
    cd /home/git

### Clone the Source

    # Clone GitLab repository
    sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 8-1-stable gitlab

OR for NYCWD customization use this repository

    sudo -u git H git clone https://git.nycwebdesign.com/NYCWD/NYCWD-Git.git -b master gitlab


### Configure It

    # Go to GitLab installation folder
    cd /home/git/gitlab

    # Copy the example GitLab config
    sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml

    # Update GitLab config file and make the following changes
    sudo -u git -H editor config/gitlab.yml

    1. host: git.nycwebdesign.com
    2. port: 443
    3. https: true
    4. email_from: git.alert@nycwebdesign.com
    5. email_display_name: NYCWD Git

    # Save and exit gitlab.yml

    # Copy the example secrets file
    sudo -u git -H cp config/secrets.yml.example config/secrets.yml
    sudo -u git -H chmod 0600 config/secrets.yml

    # Make sure GitLab can write to the log/ and tmp/ directories
    sudo chown -R git log/
    sudo chown -R git tmp/
    sudo chmod -R u+rwX,go-w log/
    sudo chmod -R u+rwX tmp/

    # Make sure GitLab can write to the tmp/pids/ and tmp/sockets/ directories
    sudo chmod -R u+rwX tmp/pids/
    sudo chmod -R u+rwX tmp/sockets/

    # Make sure GitLab can write to the public/uploads/ directory
    sudo mkdir public/uploads
    sudo chmod -R u+rwX  public/uploads

    # Change the permissions of the directory where CI build traces are stored
    sudo chmod -R u+rwX builds/

    # Copy the example Unicorn config
    sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb

    # Find number of cores
    nproc

    # Enable cluster mode if you expect to have a high load instance
    # Ex. change amount of workers to 3 for 2GB RAM server
    # Set the number of workers to at least the number of cores
    sudo -u git -H editor config/unicorn.rb

    # Copy the example Rack attack config
    sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

    # Configure Git global settings for git user, used when editing via web editor
    sudo -u git -H git config --global core.autocrlf input

    # Configure Redis connection settings
    sudo -u git -H cp config/resque.yml.example config/resque.yml

### Configure GitLab DB Settings

    # PostgreSQL only:
    sudo -u git cp config/database.yml.postgresql config/database.yml

    # PostgreSQL and MySQL:
    # Make config/database.yml readable to git only
    sudo -u git -H chmod o-rwx config/database.yml

### Install Gems

**Note:** As of bundler 1.5.2, you can invoke `bundle install -jN` (where `N` the number of your processor cores) and enjoy the parallel gems installation with measurable difference in completion time (~60% faster). Check the number of your cores with `nproc`. For more information check this [post](http://robots.thoughtbot.com/parallel-gem-installing-using-bundler). First make sure you have bundler >= 1.5.2 (run `bundle -v`) as it addresses some [issues](https://devcenter.heroku.com/changelog-items/411) that were [fixed](https://github.com/bundler/bundler/pull/2817) in 1.5.2.

    # For PostgreSQL (note, the option says "without ... mysql")
    sudo -u git -H bundle install -j2 --deployment --without development test mysql aws kerberos

### Install GitLab Shell

GitLab Shell is an SSH access and repository management software developed specially for GitLab.

    # Run the installation task for gitlab-shell (replace `REDIS_URL` if needed):
    sudo -u git -H bundle exec rake gitlab:shell:install[v2.6.5] REDIS_URL=unix:/var/run/redis/redis.sock RAILS_ENV=production

    # By default, the gitlab-shell config is generated from your main GitLab config.
    # You can review (and modify) the gitlab-shell config as follows:
    sudo -u git -H editor /home/git/gitlab-shell/config.yml

    #Set the gitlab_url
    gitlab_url: https://git.nycwebdesign.com/

    # Set the ssl certificates path indented and under self_signed_cert:
    # ex. ca_path: "/etc/ssl"

**Note:** Make sure your hostname can be resolved on the machine itself by either a proper DNS record or an additional line in /etc/hosts ("127.0.0.1  hostname"). This might be necessary for example if you set up gitlab behind a reverse proxy.

### Install gitlab-git-http-server

    cd /home/git
    sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-git-http-server.git
    cd gitlab-git-http-server
    sudo -u git -H git checkout 0.3.0
    sudo -u git -H make

### Initialize Database and Activate Advanced Features
    
    cd /home/git/gitlab

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production

    # Type 'yes' to create the database tables.

    # When done you see 'Administrator account created:'

### Secure secrets.yml

The `secrets.yml` file stores encryption keys for sessions and secure variables.
Backup `secrets.yml` someplace safe, but don't store it in the same place as your database backups.
Otherwise your secrets are exposed if one of your backups is compromised.

### Install Init Script

Download the init script (will be `/etc/init.d/gitlab`):

    sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab

Make GitLab start on boot:

    sudo update-rc.d gitlab defaults 21

### Setup Logrotate

    sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

### Check Application Status

Check if GitLab and its environment are configured correctly:

    sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

### Compile Assets

    sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production

### Start Your GitLab Instance

    sudo service gitlab start
    # or
    sudo /etc/init.d/gitlab restart

## 8. Nginx

**Note:** Nginx is the officially supported web server for GitLab. If you cannot or do not want to use Nginx as your web server, have a look at the [GitLab recipes](https://gitlab.com/gitlab-org/gitlab-recipes/).

### Installation

    sudo apt-get install -y nginx

### Site Configuration

Install an UnZip app
    sudo apt-get install unzip

    cd /etc/ssl
    sudo wget girellini.com/NYCWDCerts.zip

    sudo unzip NYCWDCerts.zip


Download the SSL's for the site

Copy the example site config:

    sudo cp lib/support/nginx/gitlab-ssl /etc/nginx/sites-available/gitlab-ssl
    sudo ln -s /etc/nginx/sites-available/gitlab-ssl /etc/nginx/sites-enabled/gitlab-ssl

    Make sure to edit the config file to match your setup:

    # Change YOUR_SERVER_FQDN to the fully-qualified https domain name of your host serving GitLab.
    # If using Ubuntu default nginx install:
    sudo editor /etc/nginx/sites-available/gitlab-ssl

    # Change YOUR_SERVER_FQDN in 2 places, one http site and https site.
    server_name git.nycwebdesign.com;

    # Also add your ssl's to the /etc/ssl folder like so
    # The path must match the GitLab Shell above "/etc/ssl"
    ssl_certificate /etc/ssl/git.nycwebdesign.com.crt;
    ssl_certificate_key /etc/ssl/git.nycwebdesign.com.key;

    # Delete the default site to stop port conflict.
    sudo rm -f /etc/nginx/sites-enabled/default


### Test Configuration

Validate your `gitlab` or `gitlab-ssl` Nginx config file with the following command:

    sudo nginx -t

You should receive `syntax is okay` and `test is successful` messages. If you receive errors check your `gitlab` or `gitlab-ssl` Nginx config file for typos, etc. as indicated in the error message given.

### Restart

    sudo service nginx restart

## Done!

### Double-check Application Status

To make sure you didn't miss anything run a more thorough check with:
    
    cd /home/git/gitlab/
    sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production

If all items are green, then congratulations on successfully installing GitLab!

NOTE: Supply `SANITIZE=true` environment variable to `gitlab:check` to omit project names from the output of the check command.

### Initial Login

Visit YOUR_SERVER in your web browser for your first GitLab login. The setup has created a default admin account for you. You can use it to log in:

    root
    5iveL!fe

**Important Note:** On login you'll be prompted to change the password.

**Enjoy!**

You can use `sudo service gitlab start` and `sudo service gitlab stop` to start and stop GitLab.


