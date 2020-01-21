# Step by Step: Getting Drupal ready to run on Lagoon

## 1. Lagoon Drupal Setting Files

In order for Drupal to work with Lagoon, we need to teach Drupal about Lagoon and Lagoon about Drupal. This happens by copying specific YAML and PHP Files into your Git repository.

You find [these Files in our GitHub repository](https://github.com/amazeeio/lagoon/tree/master/docs/using_lagoon/drupal); the easiest way is to [download these files as a ZIP file](https://minhaskamal.github.io/DownGit/#/home?url=https://github.com/amazeeio/lagoon/tree/master/docs/using_lagoon/drupal) and copy them into your Git repository. For each Drupal version and database type you will find an individual folder. A short overview of what they are:

* `.lagoon.yml` - The main file that will be used by Lagoon to understand what should be deployed and many more things. This file has some sensible Drupal defaults, if you would like to edit or modify, please check the specific [Documentation for .lagoon.yml](../lagoon-yml.md)
* `docker-compose.yml`, `.dockerignore`, and `*.dockerfile` \(or `Dockerfile`\) - These files are used to run your local Drupal development environment, they tell Docker which services to start and how to build them. They contain sensible defaults and many commented lines. iWe hope that it's well-commented enough to be self-describing. If you would like to find out more, see [Documentation for docker-compose.yml](../docker-compose-yml.md)
* `sites/default/*` - These .php and .yml files teach Drupal how to communicate with Lagoon containers both locally and in production. It also provides an easy system for specific overrides in development and production environments. Unlike other Drupal hosting systems, Lagoon never ever injects Drupal settings files into your Drupal. Therefore you can edit them however you like. Like all other files, they contain sensible defaults and some commented parts.
* `drush/aliases.drushrc.php` - These files are specific to Drush and tell Drush how to talk to the Lagoon GraphQL API in order to learn about all Site Aliases there are.
* `drush/drushrc.php` - Some sensible defaults for Drush commands.
* Add `patches` directory if you choose [drupal8-composer-mariadb](https://github.com/amazeeio/lagoon/tree/master/docs/using_lagoon/drupal/drupal8-composer-mariadb).

### Update your `.gitignore` Settings

Don't forget to make sure your `.gitignore` will allow you to commit the settings files.

Drupal is shipped with `sites/*/settings*.php` and `sites/*/services*.yml` in .gitignore, remove that, as with Lagoon we don't ever have sensitive information in the Git repository.

### Note about Webroot in Drupal 8

Unfortunately the Drupal community has not decided on a standardized webroot folder name. Some projects put Drupal within `web`, and others within `docroot` or somewhere else. The Lagoon Drupal settings files assume that your Drupal is within `web`, if this is different for your Drupal, please adapt the files accordingly.

## 2. Customise docker-compose.yml

Don't forget to customize the values in lagoon-project & LAGOON\_ROUTE with your site specific name & the URL you'd like to access the site with:

## 3. Build Images

First, we need to build the defined images:

```bash
docker-compose build
```

This will tell docker-compose to build the Docker images for all containers that have a `build:` definition in the `docker-compose.yml`. Usually for Drupal this is the case for the `cli`, `nginx` and `php`. We do this because we want to run specific **build** commands \(like `composer install`\) or inject specific environment variables \(like `WEBROOT`\) into the images.

Usually building is not needed every time you edit your Drupal code \(as the code is mounted into the containers from your host\), but rebuilding does not hurt. Plus Lagoon will build the exact same Docker images also during a deploy, you can therefore check that your build will also work during a deployment with just running `docker-compose build` again.

## 4. Start Containers

Now as the images are built, we can start the containers:

```bash
docker-compose up -d
```

This will bring up all containers. After the command is done, you can check with `docker-compose ps` to ensure that they are all fully up and have not crashed. If there is a problem, check the logs with `docker-compose logs -f [servicename]`.

## 5. Rerun `composer install` \(for Composer projects only\)

In a local development environment you most probably open the Drupal Code in your favorite IDE and you also want all dependencies downloaded and installed, so connect into the cli container and run `composer install`:

```bash
docker-compose exec cli bash
composer install
```

This might sound weird, as there was already a `composer install` executed during the build step, so let us explain:

* In order to be able to edit files on the host and have them immediately available in the container, the default `docker-composer.yml` mounts the whole folder into the the containers \(this happens with `.:/app:delegated` in the volumes section\). This also means that all dependencies installed during the Docker build are overwritten with the files on the host.
* Locally, you probably want dependencies defined as `require-dev` in `composer.json` to exist as well, while on a production deployment they would just use unnecessary space. So we run `composer install --no-dev` in the Dockerfile and `composer install` manually.

If everything went well, open the `LAGOON_ROUTE` defined in `docker-compose.yml` \(for example [http://drupal.docker.amazee.io](http://drupal.docker.amazee.io)\) and you should be greeted by a nice Drupal error. Don't worry - that's ok right now, most important is that it tries to load a Drupal site.

If you get a 500 or similar error, make sure everything loaded properly with Composer.

## 6. Check Status and Install Drupal

Finally it's time to install a Drupal, but just before that we want to make sure everything works alright. We suggest to use Drush for that:

```bash
docker-compose exec cli bash
drush status
```

This should return something like:

```bash
[drupal-example]cli-drupal:/app$ drush status
[notice] Missing database table: key_value
Drupal version       :  8.6.1
Site URI             :  http://drupal.docker.amazee.io
Database driver      :  mysql
Database hostname    :  mariadb
Database port        :  3306
Database username    :  drupal
Database name        :  drupal
PHP binary           :  /usr/local/bin/php
PHP config           :  /usr/local/etc/php/php.ini
PHP OS               :  Linux
Drush script         :  /app/vendor/drush/drush/drush
Drush version        :  9.4.0
Drush temp           :  /tmp
Drush configs        :  /home/.drush/drush.yml
                        /app/vendor/drush/drush/drush.yml
Drupal root          :  /app/web
Site path            :  sites/default
```

{% hint style="warning" %}
You may have to tell pygmy about your public key before the next step. 

If you get an error like `Permission denied (publickey)`,  check out the documentation here: [pygmy - adding ssh keys](https://pygmy.readthedocs.io/en/master/usage/#adding-ssh-keys)
{% endhint %}

Now it is time to install Drupal \(if instead you would like to import an existing SQL File, please skip to step 6, but we suggest you install a clean Drupal in the beginning to be sure everything works.\)

```bash
drush site-install
```

This should output something like:

```bash
[drupal-example]cli-drupal:/app$ drush site-install
You are about to DROP all tables in your 'drupal' database. Do you want to continue? (y/n): y
Starting Drupal installation. This takes a while. Consider using the --notify global option.
Installation complete.  User name: admin  User password: a7kZJekcqh
Congratulations, you installed Drupal!
```

Now you can visit the URL defined in `LAGOON_ROUTE` and you should see a fresh and clean installed Drupal - Congrats!

![Congrats!](https://media.giphy.com/media/XreQmk7ETCak0/giphy.gif)

## 7. Import existing Database Dump

If you have an already existing Drupal site, you probably want to import its database over to your local site.

There are many different ways to create a database dump. If your current hosting provider has Drush installed, you can use the following:

```bash
drush sql-dump --result-file=dump.sql

Database dump saved to dump.sql
```

Now you have a `dump.sql` file that contains your whole database.

Copy this file into your Git repository and connect to the CLI, and you should see the file in there:

```bash
[drupal-example]cli-drupal:/app$ ls -l dump.sql
-rw-r--r--    1 root     root          5281 Dec 19 12:46 dump.sql
```

Now you can drop the current database, and then import the dump.

```bash
drush sql-drop

drush sql-cli < dump.sql
```

Verify that everything works with visiting the URL of your project. You should have a functional copy of your Drupal site!

## 8. Drupal files directory

A Drupal site also consists of the files directory. As the whole folder is mounted into the Docker containers, just add the files into the correct folder \(probably `web/sites/default/files`, `sites/default/files` or something similar\). Remember what you've set as your webroot - [it may not be the same for all projects](step-by-step-getting-drupal-ready-to-run-on-lagoon.md#note-about-webroot-in-drupal-8).

## 9. Done!

You are done with your local setup. The Lagoon Team wishes Happy Drupalling!

If you'd like to deploy your local Drupal into Lagoon, follow the next step to get set up before you deploy: [Setting up a new project in Lagoon](../setup_project.md).
