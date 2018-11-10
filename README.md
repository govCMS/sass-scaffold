# GovCMS Project Scaffolding
**For creating new govCMS websites, and/or importing your own**


## Contents 

 - [Requirements](#req)
 - [Known issues](#issues)
 - [Pushing commits](#push)
 - [Spin up a vanilla govCMS site](#new)
 - [Importing an existing site from a backup](#imp-site)
 - [Importing a database](#imp-db)
 - [Importing public files](#imp-files)
 - [Development](#dev)
 - [Image inheritance](#img-in)
 - [Notes](#notes)
 - [Commands](#comm)
 - [@TODO](#todo)



## <a name="req"></a>Requirements

* [Docker](https://docs.docker.com/install/) - a container management system
* [pygmy](https://docs.amazee.io/local_docker_development/pygmy.html#installation) a dynamic language that compiles to JavaScript (you might need `sudo` for this depending on your ruby configuration)
* [Ahoy](http://ahoy-cli.readthedocs.io/en/latest/#installation) - optional; a shortcut manager for saving your precious fingers
* [Portainer](https://portainer.io/install.html) - optional; a nice web UI for managing Docker stuff



## <a name="issues"></a>Known issues

1. This process only applies to the [7.x-3.x branch of GovCMS](https://github.com/govCMS/govcms/tree/7.x-3.x).
2. Currently (Nov 2018), all local projects utilise the same LOCALDEV_URL. The URL used is hard-coded in. GovCMS is aware and working on a fix. To access different sites, shut down the containers of all except the one you want to see at that URL.
3. The container `test` cannot have its name changed. This prevents Drupal from being able to connect to the database for some reason.
4. When logging into the site for the first time, the 'Reset password' page does not allow resetting the password, complaining  `Password may only be changed in 24 hours from the last change`. See Step 5 of 'Spin up a vanilla govCMS site' for the workaround. 
5. The `dnsmasq` that ships with the Docker images clashes with Linux Ubuntu 16.04 LTS's default `dnsmasq` service, and throws an error when runnig `pygmy up`. Disabling Ubuntu's dnsmasq service should fix this. 
6. The mariadb database cannot be access from the `test` container via `mysql -u <username> -p ...`, and will complain of MySQL not being connected at all. **The only way to interact with the database is via Drush**. You can verify Drupal can talk to the database by running `ahoy drush status --full` and looking for the line reading `Database: connected`.



## <a name="push"></a>Pushing commits

Before pushing anything back up, you should check your local Git credentials match those used by your remote repo account. If they don't, your commits will show as originating from a different user. 

You can check your global Git user account details by inspecting `user.name` and `user.email` via:
    
        git config --list

You can then specify different user details for specific repositories using this:

        git config --local user.name '<your account username>'
        git config --local user.email <your account email>



## <a name="setup"></a>Setup

### <a name="new"></a>Spin up a vanilla govCMS site

This lets you quickly whip up a govCMS template site in Docker, but without persistent storage, i.e. **if you shut down your containers, you lose any changes made to the website**.

1. Checkout project repo and confirm the path is in Docker's file sharing config (https://docs.docker.com/docker-for-mac/#file-sharing):

        Clone with HTTPS: git clone https://projects.govcms.gov.au/dof/agency.git govcms-agency && cd $_
        Clone with SSH:   git clone git://projects.govcms.gov.au:dof/agency.git govcms-agency && cd $_

2. Start Pygmy: 

        pygmy up

    **Make sure you don't have anything running on port `80` on the host machine (like an Apache web server from XAMP, WAMP, LAMP etc, or Skype).** If you do, you may need to change the port it runs on.

3. Build and start the Docker containers:

        Ahoy:   ahoy up
        Docker: docker-compose up -d

4. Install the GovCMS Drupal profile into the new website container:

        Ahoy:   ahoy install
        Docker: docker-compose exec -T test drush si -y govcms

5. Update the user account password using Drush:

        Ahoy:   ahoy drush upwd admin --password='<your-new-password>'
        Docker: docker-compose exec -T test drush upwd admin --password='<your-new-password>'


6. Login to your newly-installed Drupal site using the domain this spits out (the full URL prompts a password reset):

        Ahoy:   ahoy login
        Docker: docker-compose exec -T test drush uli



## <a name="imp-site"></a>Importing an existing site from a backup

You can install a base govCMS site from this project, then import your files and database.



### <a name="imp-db"></a>Importing a database

1. Take a backup of the database of the site you want to import, and compress it

        tar -zcf <database-file>.sql.tar.gz </local-location/database.sql>

2. Copy it into the root directory of the `industry` project 'test' container. You can see list of the container names with `ahoy ps` or `docker-compose ps`

        docker cp <local/file.ext> <containername>:<desired-location-inside-container>

    so:

        docker cp database.sql.tar.gz industry_test_1:/app/

3. Start a new CLI session inside the `test` container, where `"$1"` is the test website container name

        docker exec -it "$1" bash,

4. Empty the database and extract + import the new one. This may take a while depending on the size of your database file.

    (Note the backticks around `drush sql-cli`; these are required, or the import doesn't run. See `drush sql-connect --help`)

        drush sql-drop -y; tar -xf database.sql.tar.gz; `drush sql-cli` < /app/database.sql

5. Exit the `test` container CLI session

        exit

6. Flush the caches

        Ahoy:   ahoy drush cc all
        Docker: docker-compose exec -T test drush cc all



### <a name="imp-files"></a>Importing public files

Currently, files can either be imported into the container, or referenced from the local file system using [Docker Volumes](https://docs.docker.com/storage/volumes/). 

1. Repeat steps 1-3 of the Database import instructions above, but with a compressed archive of your files instead.
2. Once you've jumped inside the `test` container, extract the files

        tar -xf myfiles.tar.gz -C /app/sites/default/files/

    The `owner` and `group` of the extracted files may differ from those of the container i.e. `root` > `1000`. This shouldn't matter unless you want to edit them.

3. Check the files are present and accessible by visiting one via the project site URL; `http://govcms.docker.amazee.io/sites/default/files/<example-file.pdf>`




## <a name="dev"></a>Development

* You should create your theme(s) in folders under `/themes`
* Tests specific to your site can be committed to the `/tests` folders
* The files folder is not (currently) committed
* Do not make changes to `docker-compose.yml`, `lagoon.yml`, `.gitlab-ci.yml` or the Dockerfiles under `/.docker` - these will result in your project being unable to deploy to GovCMS SaaS



## <a name="img-in"></a>Image inheritance

This project is designed to provision a Drupal 8 project onto GovCMS SaaS, using the GovCMS8 distribution, and has been prepared thus

1. The vanilla GovCMS (7.x-3.x) Distribution is available at [Github Source](https://github.com/govcms/govcms) and as [Public DockerHub images](https://hub.docker.com/r/govcms)
2. Those GovCMS images are then customised for Lagoon and GovCMS, and are available at [Github Source](https://github.com/govcms/govcmslagoon) and as [Public DockerHub images](https://hub.docker.com/r/govcmslagoon)
3. Those GovCMSlagoon images are then retrieved in this scaffold repository.


## <a name="notes"></a>Notes 
- Unless you import a database dump from another site, the out-of-the-box govCMS site will only contain the user 'admin'.
- If you import a database dump from a site where your user account is NOT an administrator, you can become one by assigning your account the administrator role using `ahoy drush urol 'administrator' <account email or user ID>`. The super admin user ID will always be '1'.


## <a name="comm"></a>Commands

Additional commands are listed in `.ahoy.yml`. You can add anything that make life easier, even motivational quotes.

* View the themes present on your site and see which are enabled
 
        ahoy drush pm-list --type=theme
        
* View the running containers for the current project

        ahoy ps

* View _all_ running containers

        docker container ls

* Remove all stopped containers from Docker (this is useful if storage space is a concern)

        docker container prune

# <a name="todo"></a>@TODO

Setting up persistent storage (your files and database)
:   - Anything you don't want to lose when shutting down the containers should live in persistent storage
   - Mention port mapping, and clashing should you already be running a local server (like a LAMP stack)

Saving your changes in new Container Images
:   - How can you capture your completed non-code work in a new Image

Connecting to the Docker container repository
:   - Saving, updating and sharing your work

Tagging Container Images
:   - For safer release management

Golden rules: Dos and don'ts of containers
:   - Something easy to consume for govCMS newcomers, and simpler than the official Docker documentation
   - Explain what kind of changes should be saved where i.e. in Container Images, the repo, local volumes etc
