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
* [Podcast](https://podcast.linuxserver.io) - on hiatus. Coming back soon (late 2018).

# PSA: Changes are happening

From August 2018 onwards, Linuxserver are in the midst of switching to a new CI platform which will enable us to build and release multiple architectures under a single repo. To this end, existing images for `arm64` and `armhf` builds are being deprecated. They are replaced by a manifest file in each container which automatically pulls the correct image for your architecture. You'll also be able to pull based on a specific architecture tag.

TLDR: Multi-arch support is changing from multiple repos to one repo per container image.

# [linuxserver/bookstack](https://github.com/linuxserver/docker-bookstack)
[![](https://images.microbadger.com/badges/version/linuxserver/bookstack.svg)](https://microbadger.com/images/linuxserverbookstack "Get your own version badge on microbadger.com")
[![](https://images.microbadger.com/badges/image/linuxserver/bookstack.svg)](https://microbadger.com/images/linuxserver/bookstack "Get your own version badge on microbadger.com")
![Docker Pulls](https://img.shields.io/docker/pulls/linuxserver/bookstack.svg)
![Docker Stars](https://img.shields.io/docker/stars/linuxserver/bookstack.svg)

[Bookstack](https://github.com/BookStackApp/BookStack) is a free and open source Wiki designed for creating beautiful documentation. Feautring a simple, but powerful WYSIWYG editor it allows for teams to create detailed and useful documentation with ease.

Powered by SQL and including a Markdown editor for those who prefer it, BookStack is geared towards making documentation more of a pleasure than a chore.

For more information on BookStack visit their website and check it out: https://www.bookstackapp.com


[![bookstack](https://s3-us-west-2.amazonaws.com/linuxserver-docs/images/bookstack-logo500x500.png)](https://github.com/BookStackApp/BookStack)

## Supported Architectures

Our images support multiple architectures such as `X86-64`, `arm64` and `armhf`. We utilise the docker manifest for multi-platform awareness. More information is available from docker [here](https://github.com/docker/distribution/blob/master/docs/spec/manifest-v2-2.md#manifest-list). 

The architectures supported by this image are:

| Architecture | Tag |
| :----: | --- |
| X86-64 | amd64-latest |
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
  -p 6875:80 \
  -v <path to data>:/config \
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
from shell: mysql -u root -p
CREATE DATABASE bookstackapp;
GRANT USAGE ON *.* TO 'myuser'@'%' IDENTIFIED BY 'mypassword';
GRANT ALL privileges ON 'bookstackapp'.* TO 'myuser'@localhost;
FLUSH PRIVILEGES;
```

Once you have completed these, you can then use the docker run command to create your BookStack container. Make sure you replace things such as <yourdbuser> with the correct data.

Then docker start bookstackapp to start the container. You should then be able to access the container at http://dockerhost:6875

Default username is admin@admin.com with password of **password**

Documentation can be found at https://www.bookstackapp.com/docs/



## Support Info

* Shell access whilst the container is running: `docker exec -it bookstack /bin/bash`
* To monitor the logs of the container in realtime: `docker logs -f bookstack`
* container version number 
  * `docker inspect -f '{{ index .Config.Labels "build_version" }}' bookstack`
* image version number
  * `docker inspect -f '{{ index .Config.Labels "build_version" }}' linuxserver/bookstack`

## Versions

* **02.07.18:** - Initial Release.
