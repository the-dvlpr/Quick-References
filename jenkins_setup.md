# Jenkins CI on OSX
Instructions on how to setup a secured Jenkins CI on a Mac.

## Download & Install dependencies
All of these operations are done with your admin user.

### Developer tools
Install the command line developer tools.

```
$ xcode-select --install
```

### Git
 - [Installer](http://sourceforge.net/projects/git-osx-installer/files/git-2.2.1-intel-universal-mavericks.dmg/download)

After installing Git, ensure that you are using it (instead of the default):

```
sudo mv /usr/bin/git /usr/bin/git-xcode
sudo ln -sf /usr/local/git/bin/git /usr/bin/git
```

### Java JDK 7

 - [Installer](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html)

### Jenkins
 - [Installer](http://mirrors.jenkins-ci.org/osx/latest)

After running the installer, visit http://localhost:8080 to see Jenkins. If it does not show up, restart the box. If it still doesn't show up, you may need to start Jenkins manually:

```
sudo launchctl load -w /Library/LaunchDaemons/org.jenkins-ci.plist
```

#### Update all plugins
Once you can access your Jenkins console,  goto `Manage Jenkins -> Manage Plugins` from the home screen.

Sometimes when you install, you will notice that the list of available plugins is empty. If that is the case, from `Advanced` tab on the `Manage Plugins` page, click on `Check now` (button available in the bottom right of the page) to forcefully check for new updates. Once that is done, you should see the list of plugins.

In the `Updates` tab, check all and click `download and install after restart`. Once downloads are finished , check `Restart Jenkins when installation is complete and no jobs are running`.

#### Install plugins

Open the `Available` tab and find the plugin entitled:

- Git Plugin
- Github plugin
- Rake plugin
- RVM plugin
- Green balls

Download and restart Jenkins.


## Jenkins user
Go into OSX `System Settings/Users`, select the `jenkins` user created by the installer and:

 - Add a secure password to it.
 - Make it admin.

All of the next operations are done with your `jenkins` user, so switch:

```
$ sudo su - jenkins
```
Proceed to install the following.

### Homebrew:
```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
$ brew doctor
$ brew update
```

### PostgreSQL
```
$ brew install postgresql
```

Ensure that it is installed:

```
$ psql --ver
psql (PostgreSQL) 9.4.1
```

Start it automatically upon login (for all users):

```
$ sudo cp /usr/local/opt/postgresql/homebrew.mxcl.postgresql.plist /Library/LaunchDaemons/
$ sudo vim /Library/LaunchDaemons/homebrew.mxcl.postgresql.plist
```

Add param at the end:

```
  <key>UserName</key>
  <string>jenkins</string>
```

Reboot your box. After reboot, create the `postgres` user:

```
$ createuser -s postgres
```

Login to PostgreSQL and grant rights to  `postgres`:

```
$ psql -d postgres
psql (9.3.5)
Type "help" for help.

postgres=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO postgres;
postgres=# ALTER USER postgres with CREATEDB;
```

Exit with `\q`.

### RVM:
```
$ \curl -sSL https://get.rvm.io | bash -s stable
$ source ~/.bash_profile
```

### Ruby
Match the version of the project:
```
$ rvm install 2.2.0
$ rvm use 2.2.0 --default
```

### Git access

As user Jenkins, create a `.ssh` directory in the Jenkins home directory.

```
$ mkdir ~/.ssh
```

Create the public private key pair. The most important thing is not to set a password, otherwise the jenkins user will not be able to connect to the git repo in an automated way.

```
$ cd ~/.ssh
$ ssh-keygen -t rsa -C "jenkins@CI"
```

Start the ssh-agent in the background and add the key:
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```

Display your newly creted public key:

```
$ cat ~/.ssh/id_rsa.pub

```
Copy the output and add it to your Git repo.

Set a git user and email address:

```
$ git config --global user.email "cloud+jenkins@example.com"
$ git config --global user.name "jenkins"
```

Connect to the Git repo. This is a one time step which will dismiss that 'Are you sure you want to connect' ssh message, again jenkins won't be able to deal with this. Just run:

```
$ ssh -T git@github.com
```

## Create Jenkins Job

### Create project
From the Jenkins home page, click on `New Item`, then select `Free-style software project` and click `OK`.

Fill in:
 - Project Name: `Myproject`
 - GitHub project: `https://github.com/myuser/myproject`
 - Under `Source Code Management`, select `Git` and fill in the repo url:
   `git@github.com:myuser/myproject.git`
 - Click `Add` next to credentials:
   - Kind: `SSH username with private key`
   - Private key: `From the jenkins master`

Click `Add` then select `jenkins` next to Credentials.

Poll SCM with a schedule of:

```
H/5 * * * *
```

Check the `Run the build in a RVM-managed environment` box, and enter in `Implementation`:

`ruby-2.2.0@myproject`

Under `Add build step`, select `Execute shell`, and enter:

```
bundle install
bundle exec rake db:setup
bundle exec rake ci:all
```

Add the "Publish JUnit test result report" post-build step in the job configuration. Enter `spec/reports/*.xml` in the `Test report XMLs` field (adjust this to suit which tests you are running).

Click on `Save`.

Click on `Build now` to start building the job, even though it will fail (no database has been initialized yet).

### Prepare & try project
```
$ cd ~/Home/jobs/myproject/workspace
$ bundle
$ rake db:setup
$ rake
```

If you get a UTF-8 error while creating the database, do the following:

```
$ psql -d postgres -W
psql (9.3.5)
Type "help" for help.

postgres=# UPDATE pg_database SET datistemplate = FALSE WHERE datname = 'template1';
```
Now we can drop it:
```
postgres=# DROP DATABASE template1;
```
Now its time to create database from template0, with a new default encoding:
```
postgres=# CREATE DATABASE template1 WITH TEMPLATE = template0 ENCODING = 'UNICODE';
```
Now modify template1 so it's actually a template:
```
postgres=# UPDATE pg_database SET datistemplate = TRUE WHERE datname = 'template1';
```
Now switch to template1 and VACUUM FREEZE the template:
```
postgres=# \c template1;
postgres=# VACUUM FREEZE;
```

Exit with `\q`.


## Configure security

### Create users
Navigate to `Manage Jenkins` and select `Configure Global Security`.  On this screen, check `Enable Security`, then under `Security Realm` select `Jenkins' own user database`. Ensure that  `Allow users to sign up` is unchecked.

Click `Save`. You will be prompted to register, add an `admin` user. Once done, you'll be automatically logged in as `admin`.

Go back to `Manage Jenkins`, you will now see an additional ` Manage Users` menu. Navigate in there, and create a `localmonitor` user.

### Add permissions
Navigate to `Manage Jenkins` and select `Configure Global Security`. On this screen, check `Project-based Matrix Authorization Strategy` under `Authorization`.

From there, add `admin` and `localmonitor` users, checking all permissions for `admin` and only `Overall Read` and `JOB read` for`localmonitor`. Save the changes.


## Add SSL with Nginx

Install Nginx:
```
$ sudo su - jenkins
$ brew install nginx
```

Start it automatically upon login (for all users):

```
$ sudo cp /usr/local/opt/nginx/homebrew.mxcl.nginx.plist /Library/LaunchDaemons/
$ sudo vim /Library/LaunchDaemons/homebrew.mxcl.nginx.plist
```

Add param at the end:

```
  <key>UserName</key>
  <string>jenkins</string>
```

If you want to sign the server with self-generated credentials, create ssl keys and cert:

```
$ mkdir /usr/local/etc/nginx/ssl
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /usr/local/etc/nginx/ssl/server.key -out /usr/local/etc/nginx/ssl/server.crt
```
Otherwise get the `server.crt` and the `server.key` from your authority.

Configure:

```
$ rm /usr/local/etc/nginx/nginx.conf
$ vim /usr/local/etc/nginx/nginx.conf
```
Paste:
```
worker_processes 4;

events {
	worker_connections 768;
}

http {
	upstream jenkins {
	  server 127.0.0.1:8080 fail_timeout=0;
	}

	server {
	  listen 4443;
	  server_name jenkins;

	  ssl on;
	  ssl_certificate /usr/local/etc/nginx/ssl/server.crt;
	  ssl_certificate_key /usr/local/etc/nginx/ssl/server.key;

	  location / {
	    proxy_set_header        Host $host;
	    proxy_set_header        X-Real-IP $remote_addr;
	    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
	    proxy_set_header        X-Forwarded-Proto $scheme;
	    proxy_redirect http:// https://;
	    proxy_pass              http://jenkins;
	  }
	}
}
```

Restart the box.
