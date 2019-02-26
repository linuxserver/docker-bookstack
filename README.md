[![linuxserver.io](https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/linuxserver_medium.png)](https://linuxserver.io)

The [LinuxServer.io](https://linuxserver.io) team brings you another container release featuring :-

 * regular and timely application updates
 * easy user mappings (PGID, PUID)
 * custom base image with s6 overlay
 * weekly base OS updates with common layers across the entire LinuxServer.io ecosystem to minimise space usage, down time and bandwidth
 * regular security updates

Find us at:
* [Discord](https://discord.gg/YWrKVTn) - realtime support / chat with the community and the team.
* [IRC](https://irc.linuxserver.io) - on freenode at `#linuxserver.io`. Our primary support channel is Discord.
* [Blog](https://blog.linuxserver.io) - all the things you can do with our containers including How-To guides, opinions and much more!
* [Podcast](https://anchor.fm/linuxserverio) - on hiatus. Coming back soon (late 2018).

# [linuxserver/bookstack](https://github.com/linuxserver/docker-bookstack)
[![](https://img.shields.io/discord/354974912613449730.svg?logo=discord&label=LSIO%20Discord&style=flat-square)](https://discord.gg/YWrKVTn)
[![](https://images.microbadger.com/badges/version/linuxserver/bookstack.svg)](https://microbadger.com/images/linuxserver/bookstack "Get your own version badge on microbadger.com")
[![](https://images.microbadger.com/badges/image/linuxserver/bookstack.svg)](https://microbadger.com/images/linuxserver/bookstack "Get your own version badge on microbadger.com")
![Docker Pulls](https://img.shields.io/docker/pulls/linuxserver/bookstack.svg)
![Docker Stars](https://img.shields.io/docker/stars/linuxserver/bookstack.svg)
[![Build Status](https://ci.linuxserver.io/buildStatus/icon?job=Docker-Pipeline-Builders/docker-bookstack/master)](https://ci.linuxserver.io/job/Docker-Pipeline-Builders/job/docker-bookstack/job/master/)
[![](https://lsio-ci.ams3.digitaloceanspaces.com/linuxserver/bookstack/latest/badge.svg)](https://lsio-ci.ams3.digitaloceanspaces.com/linuxserver/bookstack/latest/index.html)

[Bookstack](https://github.com/BookStackApp/BookStack) is a free and open source Wiki designed for creating beautiful documentation. Feautring a simple, but powerful WYSIWYG editor it allows for teams to create detailed and useful documentation with ease.

Powered by SQL and including a Markdown editor for those who prefer it, BookStack is geared towards making documentation more of a pleasure than a chore.

For more information on BookStack visit their website and check it out: https://www.bookstackapp.com


[![bookstack](https://s3-us-west-2.amazonaws.com/linuxserver-docs/images/bookstack-logo500x500.png)](https://github.com/BookStackApp/BookStack)

## Supported Architectures

Our images support multiple architectures such as `x86-64`, `arm64` and `armhf`. We utilise the docker manifest for multi-platform awareness. More information is available from docker [here](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md#manifest-list) and our announcement [here](https://blog.linuxserver.io/2019/02/21/the-lsio-pipeline-project/). 

Simply pulling `linuxserver/bookstack` should retrieve the correct image for your arch, but you can also pull specific arch images via tags.

The architectures supported by this image are:

| Architecture | Tag |
| :----: | --- |
| x86-64 | amd64-latest |
| arm64 | arm64v8-latest |
| armhf | arm32v6-latest |


## Usage

Here are some example snippets to help you get started creating a container.

### docker

```
docker create \
  --name=bookstack \
  -e PUID=1001 \
  -e PGID=1001 \
  -e DB_HOST=<yourdbhost> \
  -e DB_USER=<yourdbuser> \
  -e DB_PASS=<yourdbpass> \
  -e DB_DATABASE=bookstackapp \
  -e APP_URL=your.site.here.xyz \
  -p 6875:80 \
  -v <path to data>:/config \
  --restart unless-stopped \
  linuxserver/bookstack
```


### docker-compose

Compatible with docker-compose v2 schemas.

```
---
version: "2"
services:
  bookstack:
    image: linuxserver/bookstack
    container_name: bookstack
    environment:
      - PUID=1001
      - PGID=1001
      - DB_HOST=<yourdbhost>
      - DB_USER=<yourdbuser>
      - DB_PASS=<yourdbpass>
      - DB_DATABASE=bookstackapp
      - APP_URL=your.site.here.xyz
    volumes:
      - <path to data>:/config
    ports:
      - 6875:80
    mem_limit: 4096m
    restart: unless-stopped
```

## Parameters

Container images are configured using parameters passed at runtime (such as those above). These parameters are separated by a colon and indicate `<external>:<internal>` respectively. For example, `-p 8080:80` would expose port `80` from inside the container to be accessible from the host's IP on port `8080` outside the container.

| Parameter | Function |
| :----: | --- |
| `-p 80` | will map the container's port 80 to port 6875 on the host |
| `-e PUID=1001` | for UserID - see below for explanation |
| `-e PGID=1001` | for GroupID - see below for explanation |
| `-e DB_HOST=<yourdbhost>` | for specifying the database host |
| `-e DB_USER=<yourdbuser>` | for specifying the database user |
| `-e DB_PASS=<yourdbpass>` | for specifying the database password |
| `-e DB_DATABASE=bookstackapp` | for specifying the database to be used |
| `-e APP_URL=your.site.here.xyz` | for specifying the url your application will be accessed on (required for correct operation of reverse proxy) |
| `-v /config` | this will store any uploaded data on the docker host |

## User / Group Identifiers

When using volumes (`-v` flags) permissions issues can arise between the host OS and the container, we avoid this issue by allowing you to specify the user `PUID` and group `PGID`.

Ensure any volume directories on the host are owned by the same user you specify and any permissions issues will vanish like magic.

In this instance `PUID=1001` and `PGID=1001`, to find yours use `id user` as below:

```
  $ id username
    uid=1001(dockeruser) gid=1001(dockergroup) groups=1001(dockergroup)
```


&nbsp;
## Application Setup

This application is dependent on an SQL database be it one you already have or a new one. If you do not already have one, set up our MariaDB container.

Once the MariaDB container is deployed, you can enter the following commands into the shell of the MariaDB container to create the user, password and database that the app will then use. Replace myuser/mypassword with your own data.

**Note** this will allow any user with these credentials to connect to the server, it is not limited to localhost

```
from shell on sql container: 
mysql -u root -p
CREATE DATABASE bookstackapp;
GRANT USAGE ON *.* TO 'myuser'@'%' IDENTIFIED BY 'mypassword';
GRANT ALL privileges ON `bookstackapp`.* TO 'myuser'@'%';
FLUSH PRIVILEGES;
```

Once you have completed these, you can then use the docker run command to create your BookStack container. Make sure you replace things such as <yourdbuser> with the correct data.

Then docker start bookstackapp to start the container. You should then be able to access the container at http://dockerhost:6875

Default username is admin@admin.com with password of **password**

If you intend to use this application behind a reverse proxy, such as our LetsEncrypt container or Traefik you will need to make sure that the `APP_URL` environment variable is set, or it will not work

Documentation for BookStack can be found at https://www.bookstackapp.com/docs/

### Advanced Users (full control over the .env file)
If you wish to use the extra functionality of BookStack such as email, memcache, ldap and so on you will need to make your own .env file with guidance from the BookStack documentation.

When you create the container, do not set any arguments for any SQL settings, or APP_URL. The container will copy an .env file to /config/www/.env on your host system for you to edit. 

### Composer

Some simple docker-compose files are included for you to get started with. You will still need to manually configure the SQL server, but the compose files will get the stack running for you. 



## Support Info

* Shell access whilst the container is running: `docker exec -it bookstack /bin/bash`
* To monitor the logs of the container in realtime: `docker logs -f bookstack`
* container version number 
  * `docker inspect -f '{{ index .Config.Labels "build_version" }}' bookstack`
* image version number
  * `docker inspect -f '{{ index .Config.Labels "build_version" }}' linuxserver/bookstack`

## Updating Info

Most of our images are static, versioned, and require an image update and container recreation to update the app inside. With some exceptions (ie. nextcloud, plex), we do not recommend or support updating apps inside the container. Please consult the [Application Setup](#application-setup) section above to see if it is recommended for the image.  
  
Below are the instructions for updating containers:  
  
### Via Docker Run/Create
* Update the image: `docker pull linuxserver/bookstack`
* Stop the running container: `docker stop bookstack`
* Delete the container: `docker rm bookstack`
* Recreate a new container with the same docker create parameters as instructed above (if mapped correctly to a host folder, your `/config` folder and settings will be preserved)
* Start the new container: `docker start bookstack`
* You can also remove the old dangling images: `docker image prune`

### Via Docker Compose
* Update the image: `docker-compose pull linuxserver/bookstack`
* Let compose update containers as necessary: `docker-compose up -d`
* You can also remove the old dangling images: `docker image prune`

## Versions

* **20.01.19:** - Added php7-curl
* **04.11.18:** - Added php7-ldap
* **15.10.18:** - Changed functionality for advanced users
* **08.10.18:** - Advanced mode, symlink changes, sed fixing, docs updated, added some composer files
* **23.09.28:** - Updates pre-release
* **02.07.18:** - Initial Release.
