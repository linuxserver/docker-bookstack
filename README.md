[linuxserverurl]: https://linuxserver.io
[forumurl]: https://forum.linuxserver.io
[ircurl]: https://www.linuxserver.io/irc/
[podcasturl]: https://www.linuxserver.io/podcast/
[appurl]: https://github.com/linuxserver/Heimdall
[hub]: https://hub.docker.com/r/linuxserver/heimdall/

[![linuxserver.io](https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/linuxserver_medium.png)][linuxserverurl]

The [LinuxServer.io][linuxserverurl] team brings you another container release featuring easy user mapping and community support. Find us for support at:
* [forum.linuxserver.io][forumurl]
* [IRC][ircurl] on freenode at `#linuxserver.io`
* [Podcast][podcasturl] covers everything to do with getting the most from your Linux Server plus a focus on all things Docker and containerisation!

# linuxserver/heimdall
[![](https://images.microbadger.com/badges/version/linuxserver/heimdall.svg)](https://microbadger.com/images/linuxserver/heimdall "Get your own version badge on microbadger.com")[![](https://images.microbadger.com/badges/image/linuxserver/heimdall.svg)](https://microbadger.com/images/linuxserver/heimdall "Get your own image badge on microbadger.com")[![Docker Pulls](https://img.shields.io/docker/pulls/linuxserver/heimdall.svg)][hub][![Docker Stars](https://img.shields.io/docker/stars/linuxserver/heimdall.svg)][hub][![Build Status](https://ci.linuxserver.io/buildStatus/icon?job=Docker-Builders/x86-64/x86-64-heimdall)](https://ci.linuxserver.io/job/Docker-Builders/job/x86-64/job/x86-64-heimdall/)

Heimdall is a way to organise all those links to your most used web sites and web applications in a simple way.

Simplicity is the key to Heimdall.

Why not use it as your browser start page? It even has the ability to include a search bar using either Google, Bing or DuckDuckGo.

[![heimdall](https://raw.githubusercontent.com/linuxserver/docker-templates/master/linuxserver.io/img/heimdall-banner.png)][appurl]

## Usage

```
docker create \
--name=heimdall \
-v <path to data>:/config \
-e PGID=<gid> -e PUID=<uid>  \
-p 80:80 -p 443:443 \
-e TZ=<timezone> \
linuxserver/heimdall
```

## Parameters

`The parameters are split into two halves, separated by a colon, the left hand side representing the host and the right the container side. 
For example with a port -p external:internal - what this shows is the port mapping from internal to external of the container.
So -p 8080:80 would expose port 80 from inside the container to be accessible from the host's IP on port 8080
http://192.168.x.x:8080 would show you what's running INSIDE the container on port 80.`


* `-p 80` - The web-services.
* `-p 443` - The SSL-Based Webservice
* `-v /config` - Contains your www content and all relevant configuration files.
* `-e PGID` for GroupID - see below for explanation
* `-e PUID` for UserID - see below for explanation
* `-e TZ` - timezone ie. `America/New_York`

It is based on alpine linux with s6 overlay, for shell access whilst the container is running do `docker exec -it heimdall /bin/bash`.

### User / Group Identifiers

Sometimes when using data volumes (`-v` flags) permissions issues can arise between the host OS and the container. We avoid this issue by allowing you to specify the user `PUID` and group `PGID`. Ensure the data volume directory on the host is owned by the same user you specify and it will "just work" ™.

In this instance `PUID=1001` and `PGID=1001`. To find yours use `id user` as below:

```
  $ id <dockeruser>
    uid=1001(dockeruser) gid=1001(dockergroup) groups=1001(dockergroup)
```

## Setting up the application 

Access the web gui at http://SERVERIP:PORT

## Adding password protection

This image now supports password protection through htpasswd. Run the following command on your host to generate the htpasswd file `docker exec -it heimdall htpasswd -c /config/nginx/.htpasswd <username>`. Replace <username> with a username of your choice and you will be asked to enter a password. New installs will automatically pick it up and implement password protected access. Existing users updating their image can delete their site config at `/config/nginx/site-confs/default` and restart the container after updating the image. A new site config with htpasswd support will be created in its place.

## Info

* To monitor the logs of the container in realtime `docker logs -f heimdall`.


* container version number 

`docker inspect -f '{{ index .Config.Labels "build_version" }}' heimdall`

* image version number

`docker inspect -f '{{ index .Config.Labels "build_version" }}' linuxserver/heimdall`

## Versions

+ **06.03.18:** Use password protection if htpasswd is set. Existing users can delete their default site config at /config/nginx/site-confs/default and restart the container, a new default site config with htpasswd support will be created in its place
+ **12.02.18:** Initial Release.
