BioDesign API
=============

Server setup
------------

**Distribution** Ubuntu Server 14.04 (64 bits)

### Packages needed

* lighttpd
* python
* python-pip
* Postgresql
* PostGIS
* git

### Install Ubuntu

OVH VPS comes with [Ubuntu Server 14.04 (64 bits)](http://www.ubuntu.com/download/server)

### Fix the Locale (optional?)

Set `LANG=en_US.UTF-8` in `/etc/default/locale`, then run

    apt-get install language-pack-en-base
    update-locale

### Install postgresql and postgis

    sudo apt-get install postgresql postgresql-contrib postgis postgis-doc postgresql-9.3-postgis-2.1

Basic setup

    sudo -u postgres psql postgres
    \password postgres

### Configure postgresql

We need to create a new database and a user able to access it.

    sudo -u postgres createdb biodesign
    sudo -u postgres psql
    CREATE ROLE biodesign with LOGIN PASSWORD 'realworld'

change the password from `realworld` to whatever you want to use.

Enable postgis extension on the `biodesign` db.

    sudo -u postgres psql biodesign
    CREATE EXTENSION postgis;
    CREATE EXTENSION postgis_topology;
    CREATE EXTENSION fuzzystrmatch;
    CREATE EXTENSION postgis_tiger_geocoder;
    ALTER TABLE spatial_ref_sys OWNER TO biodesign;
    

### Install DJANGO

    sudo apt-get install python-pip libpq-dev python-dev

    sudo pip install pyscopg2
    sudo pip install flup
    sudo pip install Django

    cd /home/biodesign/biodesignapi
    ./manage.py runfcgi method=threaded host=127.0.0.1 port=3033

Add the secret key in `/etc/biodesignapi/secret.txt`.

    sudo mkdir /etc/biodesignapi
    sudo echo $SECRET_KEY > /etc/biodesignapi/secret.txt

### Install lighttpd

    sudo apt-get update
    sudo apt-get install lighttpd

Create the configuration file `/etc/lighttpd/conf-available/api.biodesign.cc.conf` with following content

    #
    # django via fcgi
    #
    $HTTP["host"] == "api.biodesign.cc" {
      server.document-root = "/home/biodesign/biodesignapi/"
      fastcgi.server = (
          "/biodesignapi.fcgi" => (
            "main" => (
              # Use host / port instead of socket for TCP fastcgi
              "host" => "127.0.0.1",
              "port" => 3033,
              "check-local" => "disable",
              "allow-x-send-file" => "enable",
              )
            ),
          )
      accesslog.filename = "/var/log/lighttpd/biodesignapi/access.log"
      dir-listing.activate = "disable"
      server.errorfile-prefix = "/var/www/biodesignapi/static/"

      alias.url = (
        "/static" => "/var/www/api/static/",
        # this will need to change later
        "/media" => "/home/biodesign/biodesignapi/cplanner/uploads/",
      )

      url.rewrite-once = (
        "^(/static.*)$" => "$1",
        "^(/media/.*)$" => "$1",
        "^/\d+\.\d+\.\d+(/media/.*)$" => "$1",
        "^(/.*)$" => "/biodesignapi.fcgi$1",
      )
    }


Activate the newly created configuration

    sudo ln -s /etc/lighttpd/conf-available/10-fastcgi.conf /etc/lighttpd/conf-enabled/10-fastcgi.conf
    sudo ln -s /etc/lighttpd/conf-available/10-accesslog.conf /etc/lighttpd/conf-enabled/10-accesslog.conf
    sudo ln -s /etc/lighttpd/conf-available/api.biodesign.cc.conf /etc/lighttpd/conf-enabled/api.biodesign.cc.conf

Check the syntax is ok

    lighttpd -t -f /etc/lighttpd/lighttpd.conf

### Setup HTTPS

### Bonus: setup a git repository for automatic deployement

### Install GIT

    sudo apt-get install git

Now we want to create a user that can push to git through
SSH without having shell access. [tutorial](http://pagesofinterest.net/blog/2011/06/secure-git-server-preventing-git-user-logging-in-via-ssh/)

Create a new git user

    adduser git

Create the `.ssh` directory for this user.

    mkdir /home/git/.ssh

Copy the ssh public key of the remote client into `/home/git/.ssh/authorized_keys`.

Add the git repo to `/home/git`

    cd /home/git
    git clone <git_repo_url>
    sudo chown -R git:git <git_repo_url>

#### Create a bare repository

Setup **new**  repository on server (e.g. in `/home/biodesign`)

    GIT_DIR=BioDesignAPI.git git init  
    cd BioDesignAPI.git  
    git --bare update-server-info  
    cp hooks/post-update.sample hooks/post-update

Setup **current** BioDesignAPI repo

    git clone --bare <url>
    cd BioDesignAPI.git  
    git --bare update-server-info  
    cp hooks/post-update.sample hooks/post-update

#### Git over SSH

The repo can be accessed through SSH by adding the remote. For example:

    ssh://biodesign@10.0.0.123/home/biodesign/BioDesignAPI.git

#### Automatic deployment on push

Create a writable repository in the webserver.

    mkdir ~/BioDesignAPI_deployed
    sudo ln -s ~/BioDesignAPI_deployed/ /var/www/html/biodesign

Create the hook script

    cd ~/BioDesignAPI.git
    cat > hooks/post-receive
    #!/bin/sh
    GIT_WORK_TREE=/var/www/www.example.org git checkout -f
    $ chmod +x hooks/post-receive

Optionally, config git user and email:

    git config --global user.email "your@email.com"
    git config --global user.name "Your Name"


