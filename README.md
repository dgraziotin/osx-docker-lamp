#osx-docker-lamp, a.k.a dgraziotin/lamp
As of [Docker for Mac v1.12.0](https://docs.docker.com/engine/installation/mac/), there is no need for using my Docker images anymore. Thanks for the support!

    Out-of-the-box LAMP+phpMyAdmin Docker image that *just works* on Mac OS X. 
    Including write support for mounted volumes (Website, MySQL).
    No matter if using the official boot2docker or having Vagrant in the stack, as well.

osx-docker-lamp, which is known as 
[dgraziotin/lamp](https://registry.hub.docker.com/u/dgraziotin/lamp/) 
in the Docker Hub, is a fork of 
[tutumcloud/tutum-docker-lamp](https://github.com/tutumcloud/tutum-docker-lamp), 
which is an "Out-of-the-box LAMP image (PHP+MySQL) for Docker". 

osx-docker-lamp does what tutumcloud/tutum-docker-lamp, plus:

- It is based on [phusion/baseimage:latest](http://phusion.github.io/baseimage-docker/)
  instead of ubuntu:trusty.
- It works flawlessy regardless of using boot2docker standalone or with Vagrant. You will need to set three enrironment varibles, though.
- It fixes OS X related [write permission errors for Apache](https://github.com/boot2docker/boot2docker/issues/581)
- It lets you mount OS X folders *with write support* as volumes for
  - The website
  - The database
- If `CREATE_MYSQL_BASIC_USER_AND_DB="true"`, it creates a default database and user with permissions to that database
- It provides phpMyAdmin at /phpmyadmin
- It is documented for less advanced users (like me)


##Usage

    If using Vagrant, please see the extra steps in the next subsection.

If you need to create a custom image `youruser/lamp`, 
execute the following command from the `osx-docker-lamp` source folder:

	docker build -t youruser/lamp .

If you wish, you can push your new image to the registry:

	docker push youruser/lamp

Otherwise, you are free to use dgraziotin/lamp as it is provided. Remember first
to pull it from the Docker Hub:

    docker pull dgraziotin/lamp

###Vagrant

If, for any reason, you would rather use Vagrant (I suggest using [AntonioMeireles/boot2docker-vagrant-box](https://github.com/AntonioMeireles/boot2docker-vagrant-box)), you need to add the following three variables when running your box:

-`VAGRANT_OSX_MODE="true"` for enabling Vagrant-compatibility
-`DOCKER_USER_ID=$(id -u)` for letting Vagrant use your host user ID for mounted folders
-`DOCKER_USER_GID=$(id -g)` for letting Vagrant use your host user GID for mounted folders

See the Environment variables section for more options.

###Running your LAMP docker image

If you start the image without supplying your code, e.g.,

	docker run -t -i -p 80:80 -p 3306:3306 --name osxlamp dgraziotin/lamp

At http://[boot2docker ip, e.g., 192.168.59.103] you should see an 
"Hello world!" page.

At http://[boot2docker ip]/phpmyadmin you should see a running phpMyAdmin instance.


###Loading your custom PHP application

In order to replace the _Hello World_ application that comes bundled with this 
docker image, my suggested layout is the following:

- _Project name_ folder
  - app subfolder
  - mysql subfolder (optional)

The app folder should contain the root of your PHP application.

Run the following code from within the _Project name_ folder.

	docker run -i -t -p "80:80" -p "3306:3306" -v ${PWD}/app:/app --name yourwebapp dgraziotin/lamp

Test your deployment:

	http://[boot2docker ip]
	http://[boot2docker ip]/phpmyadmin

If you wish to mount a MySQL folder locally, so that MySQL files are saved on your
OS X machine, run the following instead:

	docker run -i -t -p "80:80" -p "3306:3306" -v ${PWD}/mysql:/var/lib/mysql -v ${PWD}/app:/app --name yourwebapp dgraziotin/lamp

The MySQL database will thus become persistent at each subsequent run of your image.

##Environment description


###The /app folder

Apache is configured to serve the files from the `/app` folder, which is a symbolic
link to `/var/www/html`. In osx-docker-lamp, the apache user `www-data` 
has full write permissions to the `app` folder.

###Apache

Apache is pretty much standard in this image. It is configured to serve the Web app
at `app` as `/` and phpMyAdmin as `/phpmyadmin`. Mod rewrite is enabled.
To serve the Web app from a directory within `/app/` set the APACHE_ROOT environment variable. ie. if `APACHE_ROOT=web` then `/app/web` will be served by apache.

Apache runs as user www-data and group staff. The write support works because the
user www-data is configured to have the same user id as the one employed by boot2docker (1000).

###phpMyAdmin

The latest version of phpMyAdmin is grabbed from sourceforge and installed in
the folder `/var/www/phpmyadmin`. 

PhpMyAdmin can be reached from 
http://[boot2docker ip]/phpmyadmin. Only the users `admin` and `user` can access
phpMyAdmin.

At your convenience, a not-so-random blowfish_secret is stored in phpMyAdmin 
configuration, which is at `/var/www/phpmyadmin/config.inc.php`

###MySQL

MySQL runs as user www-data, as well. This is not the best settings for production.
However, this is needed for proving write support to mounted volumes under Mac OS X.

####The three MySQL users

The bundled MySQL server has two users, that are `root` and `admin`, and an optional
third user `user`.

The `root` account comes with an empty password, and it is for local connections
(e.g., using some code). The `root` user cannot remotely access the database 
(and the container).

However, the first time that you run your container, a new user `admin` 
with all root privileges  will be created in MySQL with a random password. 

To get the password, check the logs of the container by running:

	docker logs [name or id, e.g., mywebsite]

You will see an output like the following:

	========================================================================
	You can now connect to this MySQL Server using:

	    mysql -uadmin -p47nnf4FweaKu -h<host> -P<port>

	Please remember to change the above password as soon as possible!
	MySQL user 'root' has no password but only allows local connections
	========================================================================

In this case, `47nnf4FweaKu` is the password allocated to the `admin` user.

Finally, an optional a user called `user` with password `password` can be created for your convenience either when:
 - The environment variable `CREATE_MYSQL_BASIC_USER_AND_DB` is true; or
 - Any of the `MYSQL_USER_*` variable (explained below) is true
The user is called `user` and has as password `password`.

The `user` user has full privileges on a database called `db`, which is also created
for your convenience. As with the `admin` user, the user `user` can access
the MySQL server from any host (`%`).
The user name, password, and database name can be changed using
the the `MYSQL_USER_*` variables, explained below.

##Environment variables

- `APACHE_ROOT` tells Apache which folder within the app volume so serve as the web root.
- `MYSQL_ADMIN_PASS="mypass"` will use your given MySQL password for the `admin`
user instead of the random one.
- `CREATE_MYSQL_BASIC_USER_AND_DB="true"` will create the user `user` with db `db` and password `password`. Not needed if using one of the following three `MYSQL_USER_*` variables
- `MYSQL_USER_NAME="daniel"` will use your given MySQL username instead of `user`
- `MYSQL_USER_DB="supercooldb"` will use your given database name instead of `db`
- `MYSQL_USER_PASS="supersecretpassword"` will use your given password  instead of `password`
- `PHP_UPLOAD_MAX_FILESIZE="10M"` will change PHP upload_max_filesize config value
- `PHP_POST_MAX_SIZE="10M"` will change PHP post_max_size config value
- `VAGRANT_OSX_MODE="true"` for enabling Vagrant-compatibility
- `DOCKER_USER_ID=$(id -u)` for letting Vagrant use your host user ID for mounted folders
- `DOCKER_USER_GID=$(id -g)` for letting Vagrant use your host user GID for mounted folders

Set these variables using the `-e` flag when invoking the `docker` client.

	docker run -i -t -p "80:80" -p "3306:3306" -v ${PWD}/app:/app -e MYSQL_ADMIN_PASS="mypass" --name yourwebapp dgraziotin/lamp

Please note that the MySQL variables will not work if an existing MySQL volume is supplied.
